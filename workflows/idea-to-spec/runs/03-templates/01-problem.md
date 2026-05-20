# 01 — Problem

## Problem

`runs/02-full-workflow/07-spec.md` defines 12 artifact types for the spec-to-code workflow plus 5 knowledge-base files. Every artifact has an inline template sketch in the Appendix. Those sketches are not usable standalone:

- They are buried in a 600-line document — an executor invoking a single node would need to locate the right section.
- They contain no quality criteria: what does a well-filled section look like? What does a thin or wrong one look like?
- They contain no "do not" rules: common failure modes an executor should avoid.
- They do not state prerequisites within the artifact itself (e.g., "this node only runs after Gate X is APPROVED").
- `issue.md` has no template at all — the spec flags this as open question #5.
- The knowledge-base template is generic across all 5 roles, giving each executor no role-specific guidance.

## Who has it

Any executor (Claude Code, opencode, pi, or human) that is invoked for a single node in the workflow. They receive the template file for their node's output artifact. If the template is inadequate, the artifact will be thin, missing sections, or structurally wrong.

## Why now

Before the workflow YAML can be written, every node's `outputs` declaration must reference a template file. Without template files, the validator has nothing to check format against, and the orchestrator has nothing to pass to executors.

## Success criteria

- One template file per artifact type: 12 workflow artifacts + 1 issue input + 5 knowledge-base files = **18 template files**.
- Each template file is self-contained: an executor reading only that file knows their role, what to read, what to write, and what constitutes a good output.
- Templates are validated by: (a) section existence checks, and (b) sentinel checks (`APPROVED`/`BLOCKED` first-line) where applicable.
- `issue.md` template exists and is simple enough for a human to fill in a few minutes.
- Knowledge base templates are role-specific: each includes the role's default prompt stub and role-specific heuristics format.

## Non-goals

- Writing the actual content of any knowledge-base file (templates define structure; content accumulates over runs).
- Writing the workflow YAML (`spec-to-code.yaml`).
- Defining executor prompts (prompts invoke the templates; templates come first).
- Filling any template with example content (that is a follow-up validation run).

## Constraints

- Templates are markdown files.
- Template instruction sections must be visually distinct from fillable sections — an executor must not accidentally copy instructions into their output.
- Templates must be small enough to fit in an executor's context alongside the node's role prompt — target single-page per template where possible.
- Section names must be stable identifiers: the validator checks for them by name.
