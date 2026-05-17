# 01 — Problem

## Problem

The proposed role set (ba / architect / test-arch / developer / reviewer / tester / meta) for a software-building workflow needs validation and concrete artifact templates before it can be wired into `spec-to-code` as YAML node definitions.

## Who has it

The developer building the `spec-to-code` workflow (see `runs/00-idea/07-spec.md`). They need a vetted role taxonomy and one fillable template per role so workflow nodes can produce predictable, validatable artifacts.

## Why now

`07-spec.md` declares that each node has a `role`, `prompt`, and `outputs` field. Until the roles and their artifact contracts are pinned down, the YAML cannot be written and the artifact validator has nothing to validate against. This run produces the contents.

## Success criteria

- A final role list — kept, removed, added — with one-sentence justification per role.
- For each kept role: an artifact contract (purpose, scope boundaries, what it does NOT do) AND a fillable content template with named sections.
- Templates are concrete enough that an executor (Claude Code / opencode / pi) can fill them in without further prompting beyond the role's stage prompt.
- Pipeline shape is confirmed or revised — the fan-out (`architect → test-arch || developer`) and the reviewer/tester separation are explicitly evaluated.

## Non-goals

- Producing the actual `spec-to-code.yaml` file (that's a follow-up; this run produces inputs *for* it).
- Implementation work on `flowctl` (the orchestrator from run 00).
- Solving how each role's prompt is written (templates pin the *output* shape; prompts come later).

## Constraints

- Templates must be markdown (the framework's artifact format).
- Templates must be small enough to fit comfortably in an executor's context — single page where possible.
- Each template's sections must be validatable by simple existence-and-non-empty checks; no semantic validation in v1.
- Role set must work for "solo dev + AI agents" — that was the persona established in run 00.

## Open questions for downstream stages

1. Is `meta` part of the workflow or out-of-band? Affects whether `reflect.md` is a node output or a separate retrospective.
2. Is `test-arch` worth being a distinct role, or should test design fold into `architect`?
3. How are roles assigned to executors (`claude-code` / `opencode` / `pi` / human)? Defer to `spec-to-code` config or declare per-role here?
4. Should the templates include filled-in *examples* for each section, or only empty headings? Examples bloat the template but help executors stay on contract.
