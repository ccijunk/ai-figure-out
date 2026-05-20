# 02 — Context

## What already exists

### Inline templates in `07-spec.md`

The spec Appendix has markdown code blocks for 12 artifacts. Each block shows section headers and one-line guidance in square brackets. They establish section names and rough content shape but nothing more. They are the starting point, not the finish line.

### `idea-to-spec` agent definitions (`agents/01-06.md`)

The closest working model of self-contained role/artifact contracts in this repo. Each agent file has:
- **Role** — persona and mindset.
- **Input** — which files to read.
- **Output** — artifact name + required sections, named explicitly.
- **Behavior** — imperative do/do-not rules specific to this role.

These four fields are the minimum a template needs to be self-contained. The agent files are the direct model to follow.

### `runs/01-artifact/07-spec.md`

Produced per-role artifact contracts (purpose, scope, NOT-DO). Those contracts are now subsumed by the richer 02-full-workflow spec, but the language used there ("does NOT write code", "does NOT design tests") is the right register for "do not" rules in templates.

---

## Key design questions this context answers

### How should instructions be separated from fillable content?

Three precedents:
1. **HTML comments** (`<!-- ... -->`): invisible in rendered markdown, visible in raw. Used in many static-site generators for front-matter alternatives. Downside: some editors/renderers strip them.
2. **Frontmatter block** (`---` YAML): standard in Jekyll, Hugo, Obsidian. Downside: not all executors handle YAML frontmatter; adds a parsing requirement.
3. **Top-level `## Instructions` section followed by `---` divider**: pure markdown, always visible, always preserved. Executor is told explicitly to omit this section from their output. Downside: executor may include it anyway if the instruction is missed.

The `idea-to-spec` agents use option 3 implicitly (their full file is instructions; they produce a separate output file). For templates where the executor fills the file in-place, an HTML comment block is the cleanest separation — instructions are present for the executor reading raw markdown but absent in rendered output.

### What does "quality criteria" look like in a template?

The `agents/05-critic.md` shows the right model: it lists what to "look hardest at" — hidden assumptions, unfalsifiable claims, unmeasurable success criteria. Template quality criteria should follow the same pattern: name the failure mode, not the ideal.

### What artifacts are gate sentinels vs. freeform?

From 07-spec.md:
- **Sentinel artifacts** (first line must be `APPROVED` or `BLOCKED`): `domain-review.md`, `testability-review.md`, `code-review.md`, `test-review.md`.
- **Freeform artifacts**: `clarify.md`, `design.md`, `test-design.md`, `implementation.md`, `test-implementation.md`, `final-review.md`, `reflect.md`.
- **Machine-generated**: `test-results.md` (format spec, not executor-filled).
- **Human-filled**: `issue.md` (simple; must not require technical knowledge).
- **Accumulating**: `knowledge-base/<role>.md` (append-only; structured but grows over time).

Sentinel templates need stronger formatting enforcement: the validator rejects the artifact if the sentinel is absent or mis-cased.

### How many files is 18?

- 4 sentinel artifacts
- 7 freeform artifacts
- 1 machine-generated artifact
- 1 human-filled input
- 5 knowledge-base files

Total: 18. Manageable as a flat directory (`templates/` under the workflow root).

---

## Constraints discovered

- **Executor context budget.** A template + role prompt + referenced inputs must fit in a single context window. Templates must be lean. The spec appendix templates average ~15 lines each — that is the right size.
- **Section names are validator anchors.** Renaming a section in a template is a breaking change: it breaks the validator and any existing artifacts. Section names should be chosen carefully and documented as stable identifiers.
- **Machine-generated `test-results.md` is different.** The executor for this node is a test runner, not an LLM. Its template is a format specification for the runner's output, not a fillable guide. It should be clearly marked as such.
- **`issue.md` is human-authored, not agent-authored.** Its template must be simple and jargon-free. A developer should be able to fill it in 5 minutes without reading the spec.
