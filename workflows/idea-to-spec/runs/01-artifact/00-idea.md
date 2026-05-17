# Idea

Validate / critique a proposed set of roles and their output artifacts for a software-building workflow.

Target use: roles + artifacts will be the nodes and `outputs` declarations in a `spec-to-code` workflow (see `workflows/idea-to-spec/runs/00-idea/07-spec.md`).

## Proposed roles and artifacts

| Role | Output file |
|---|---|
| ba | `clarify.md` |
| architect | `design.md` |
| test-arch | `test-design.md` |
| developer | `implementation.md` |
| reviewer | `review.md` |
| tester | `test-report.md` |
| meta | `reflect.md` |

## What I want from this workflow run

- Is this the right set of roles for a software-building workflow, or are roles missing / redundant / mis-scoped?
- Are the artifact contracts (sections, severity, what each role does and doesn't do) well-defined?
- Does the pipeline shape work — especially the fan-out (architect → test-arch || developer) and the adversarial separation between reviewer and tester?
- Should `meta` be in the workflow or outside it?
- output:
    1. role, 
    2. for each role, what is its artifact 
        - including the format and content which is an a tempate



## Context

Already covered in prior conversation:
- Per-role contracts proposed (sections per artifact).
- Two pushbacks already noted: review/test overlap risk; design.md must be stable before fan-out.
