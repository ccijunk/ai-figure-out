# 03 — Options

Three approaches to template structure. Each makes a different bet on what the executor needs and where instructions belong.

---

## Option A — Thin templates (sections only)

**Pitch:** Expand the inline spec appendix into separate files. Each file is the fillable body only — section headers plus one-line guidance in brackets. No separate instruction block.

**Sketch:**
- 18 files, each 10–20 lines.
- Identical to the spec appendix blocks, just split out.
- Role, inputs, and prerequisites live in the workflow YAML / role prompt, not the template.
- Validator checks section existence and non-emptiness.

**Tradeoffs:** Minimal overhead. Templates are small. Easy to read and maintain. But: executor has no role context, no quality criteria, no "do not" rules from the template itself. They must read the spec and the role prompt separately to understand what they're doing. An executor invoked in isolation — with only the template — will produce something structurally correct but potentially wrong in content.

**Biggest risk:** Executors produce thin, compliant-but-useless output because the template gave them no signal about quality. "Non-empty" is not the same as "useful."

---

## Option B — Self-describing templates (instructions embedded as comments)

**Pitch:** Each template has an HTML comment block at the top with: role, inputs, prerequisites, quality criteria, and do-not rules. Below the comment is the fillable body. The comment is invisible in rendered markdown and unambiguously not part of the output.

**Sketch:**
- 18 files. Each starts with a `<!-- ... -->` block (4–8 lines of metadata).
- Fillable body follows — same section headers as Option A.
- Validator checks sections in the body; comment block is ignored by the validator.
- Executor reading raw markdown sees full context. Executor outputting rendered markdown produces clean output.

**Tradeoffs:** Self-contained for any executor that reads raw markdown. Adds ~6 lines per file. Comment block is visually distinct — cannot be confused with fillable content. But: HTML comments are stripped by some markdown renderers. If an executor only sees rendered output (e.g., a web UI), the instructions are invisible. Also: comment block must be kept in sync with the spec if role contracts change.

**Biggest risk:** Executor's environment strips HTML comments. Instructions disappear silently; executor is back to Option A behavior with no warning.

---

## Option C — Two-section templates (visible instruction header + fillable body)

**Pitch:** Each template has two explicit sections separated by a horizontal rule. Top: a `## Template guide` section with role, inputs, prerequisites, quality criteria, do-not rules. Bottom: the fillable body starting at `## <first section>`. The instruction section is visible in all renderers. Executor is told explicitly: "Do not include the Template guide section in your output — start from the first numbered section."

**Sketch:**
- 18 files. Top section: `## Template guide` (5–10 lines). Divider: `---`. Bottom: fillable sections.
- Role prompt instructs executor: output starts at the `---` divider; omit the guide.
- Validator checks only sections below the divider.
- Both executor and human reviewer can read the guide without a raw-markdown viewer.

**Tradeoffs:** Always visible, renderer-independent. Guide is easy to update without touching section names. Instruction to "start from the divider" must be in the role prompt — if it's missing, executor may include the guide in output. But this failure is detectable (the guide section name appears in the output; validator can check for its absence). Slightly more per-file overhead than Option B.

**Biggest risk:** Executor includes the `## Template guide` section in their output. Mitigation: validator rejects any output that contains the string `## Template guide`.

---

## How the options differ

| | A | B | C |
|---|---|---|---|
| Instructions visible in renderer | ✗ | ✗ | ✓ |
| Instructions visible in raw markdown | (none) | ✓ | ✓ |
| File size | smallest | small | medium |
| Executor failure mode | thin output | stripped instructions | included guide |
| Failure detectable by validator | no | no | yes |
| Requires raw-markdown renderer | no | yes | no |
| Sync burden when spec changes | low | medium | medium |
