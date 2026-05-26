# 03 — Options

Three approaches to defining the artifact schemas, differing in how they handle the gaps and the human-gate artifacts.

---

## Option A — Minimal delta from run 03

**Pitch:** Reuse all 13 run 03 schemas unchanged. Add schemas only for the net-new artifacts (`verdict.txt`, script outputs, `reviewer` memory). Leave `reject-reason.txt` gap as an open question — remove it from the input declarations in the YAML as a cleanup step.

**Sketch:**
- `clarify.md`, `design.md`, `test-design.md`, `implementation.md`, `test-results.md`, `code-review.md`, `test-review.md`, `final-review.md`, `reflect.md`, 5 memory files: carry forward from run 03 verbatim.
- Human gate `.md` artifacts (`domain-review.md`, `testability-review.md`) lose their sentinel class — they become simple freeform review documents since the `verdict.txt` carries the routing signal.
- Add: `verdict.txt`, `requirement.md`, `repo-root.txt`, `branch-name.txt`, `pr-url.txt`, `workflow:memory/reviewer.md`.
- Remove `reject-reason.txt` from YAML input declarations silently.

**Tradeoffs:** Low effort. Schemas proven across prior work. Loses signal on `reject-reason.txt` gap — human reviewers don't get structured feedback injected into retry nodes (they only get the full review `.md`). Human gate `.md` artifacts become looser without the sentinel obligation.

**Biggest risk:** `reject-reason.txt` removal means retry nodes (`ba`, `architect`, `developer`, `test_developer`) lose their structured feedback input — they get the full review `.md` injected but there's no focused rejection reason. Retry quality degrades silently.

---

## Option B — Human gate owns both verdict and rejection reason

**Pitch:** Redesign the human gate output contract so the human writes `verdict.txt` (APPROVED/BLOCKED, bare) AND the review `.md` contains a `## Rejection reason` section that is injected as the retry node's feedback. Remove `reject-reason.txt` as a separate file — it was a redundant second channel.

**Sketch:**
- Human gate `.md` artifacts get a mandatory `## Rejection reason` section (write "N/A" if APPROVED).
- Retry node prompts receive the review `.md` as the feedback input, relying on its `## Rejection reason` section rather than a separate file.
- `reject-reason.txt` removed from the YAML entirely — closing the gap by eliminating the artifact.
- All AI-node schemas from run 03 carry forward with minor adjustments for the human-gate context.
- Script artifacts (`requirement.md`, `repo-root.txt`, etc.) get explicit format specs.

**Tradeoffs:** Clean gap resolution — no dangling unproduced artifact. Human is guided to write a focused rejection reason, which improves retry quality. One fewer file per run. Requires updating the YAML `inputs` declarations for `ba`, `architect`, `developer`, `test_developer` to reference the review `.md` instead of `reject-reason.txt`.

**Biggest risk:** Human reviewers filling in `## Rejection reason` under time pressure write thin or generic text. The section exists but provides no more signal than reading the full review `.md`. The fix (structured rejection reason) depends on human discipline.

---

## Option C — Explicit rejection reason as a separate human-written file; keep YAML as-is

**Pitch:** Keep `reject-reason.txt` as a separate, minimal human-written file with a strict format: one or two sentences, plaintext, no headers. Its purpose is not a full review but a focused machine-readable (or prompt-injectable) rejection summary. The full review `.md` remains for context.

**Sketch:**
- `reject-reason.txt` gets an explicit schema: plaintext, 1–3 lines, no headers, required when `verdict.txt` is `BLOCKED`, empty (or absent) when APPROVED.
- Human gate nodes produce three artifacts: `verdict.txt`, `reject-reason.txt`, review `.md`.
- YAML `inputs` for retry nodes remain unchanged — they already declare `reject-reason: reject-reason.txt`.
- Script artifacts and memory files get explicit schemas.
- Human gate review `.md` artifacts become simpler since `reject-reason.txt` carries the focused signal.

**Tradeoffs:** No YAML changes needed — the existing topology is preserved. Retry nodes get a tight, purposeful input. Separation of concerns: `verdict.txt` routes, `reject-reason.txt` explains, review `.md` documents. Human has three things to fill — slightly more friction per review cycle.

**Biggest risk:** Three separate outputs per human gate increases friction. A human doing a review under time pressure fills `verdict.txt`, skips `reject-reason.txt`, and the retry node gets an empty input without error (unless the orchestrator validates presence of `reject-reason.txt` when verdict is BLOCKED).

