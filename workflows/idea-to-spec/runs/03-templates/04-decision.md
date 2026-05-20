# 04 — Decision

## Chosen approach

**Option C — two-section templates (visible instruction header + fillable body).**

## Rationale

Option A produces compliant but thin output — it has no way to tell the executor what "good" looks like.

Option B hides instructions from rendered-markdown viewers. Most LLM-based executors receive rendered output from the orchestrator, not raw markdown. There is no guarantee raw markdown is passed through. Instructions silently disappearing is the worst failure mode: undetectable, no warning.

Option C keeps instructions visible in all rendering environments. The only failure mode — executor includes the `## Template guide` section in output — is detectable: the validator rejects any artifact containing that exact heading. The extra 5–10 lines per file is acceptable overhead given 18 files.

## Template file structure

Every template file follows this layout:

```
## Template guide
role:        <which role fills this>
node:        <node ID in the workflow>
inputs:      <files this node reads before filling this template>
prerequisite: <what gate must be APPROVED before this node runs, or "none">
sentinel:    <yes / no — whether first line of first fillable section must be APPROVED or BLOCKED>
quality:     <one sentence on what a well-filled artifact achieves>
do-not:
  - <failure mode 1>
  - <failure mode 2>
  - <failure mode 3>

---

## <First fillable section>
...
```

Rules:
- `## Template guide` heading is a reserved word — the validator rejects any output containing it.
- The `---` divider is the start-of-output boundary. Everything above it is instruction; everything below is the artifact.
- For sentinel artifacts, the first line of the first fillable section must be exactly `APPROVED` or `BLOCKED` (case-sensitive). Validator checks this.
- For machine-generated artifacts (`test-results.md`), the "Template guide" still exists but the role is `runner (automated)` and the body is a format spec, not a fillable guide.

## File locations

All 18 template files live at:

```
workflows/spec-to-code/templates/
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
  knowledge-base/ba.md
  knowledge-base/architect.md
  knowledge-base/test-arch.md
  knowledge-base/developer.md
  knowledge-base/test-developer.md
```

## Open questions for the Spec Writer

1. Knowledge-base templates have a dual purpose: they are both the template for initial creation AND the file that gets appended to over runs. Should the template guide be stripped out before the file is used as a live knowledge-base file, or does the guide stay in the file permanently?
2. `test-results.md` is machine-generated. Should its template be in the same `templates/` directory as the executor-facing templates, or in a separate `schemas/` location to signal its different nature?
