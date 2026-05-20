# Idea

Every `.md` artifact mentioned in `runs/02-full-workflow/07-spec.md` needs a standalone template file.

The templates in the spec are inline sketches — useful for reading the spec, not useful for an executor that needs to fill one in. A standalone template should tell the executor everything it needs: what role it is, what inputs to read, what each section must contain, and what "done" looks like — without requiring the executor to read the full spec.

## What I want from this run

- One template file per artifact in the spec-to-code workflow.
- Each template is self-contained: role, inputs, prerequisites, section guidance, common failure modes.
- `issue.md` gets a template (currently absent from the spec — open question #5 in 07-spec.md).
- Knowledge base files get role-specific templates, not one generic template.
- The output of this run is the complete content of every template file, ready to copy out.

## Source

`runs/02-full-workflow/07-spec.md` — the workflow spec being templated.
