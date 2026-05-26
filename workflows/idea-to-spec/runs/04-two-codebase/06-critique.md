# 06 — Critique

## 1. `cwd = run_dir` for analysis nodes is wrong

**What:** "Analysis nodes (ba, architect, test-arch, reviews, reflect) invoke opencode with `cwd = run_dir`."

**Why it's wrong:** `run_dir` is inside the workflow codebase (`.flows/runs/<id>/`). Analysis nodes produce `.md` artifacts in that directory — that is fine. But their `cwd` being `run_dir` means opencode, when invoked, would operate inside the workflow codebase's run directory. This is correct for artifact production, but it creates a subtle problem: if any analysis node needs to READ from `repo:` (e.g., architect reading existing domain docs in the target repo), the tool's file access pattern is `run_dir`-rooted and `repo:` paths require explicit prefix handling. The spec says "`cwd = run_dir`" but doesn't explain how `repo:` reads work in that context. This is not a blocker but it is hand-wavy.

**Severity:** `major`

---

## 2. "Two flowctl extensions" buries a real contribution scope question

**What:** "Implement `on_blocked` retry primitive in flowctl. Implement `append` output mode for `workflow:` files in flowctl." Listed as Milestones 3 and 4.

**Why it's wrong:** These are changes to `flowctl` — an external repo the user does not own. The spec treats them as implementation tasks without addressing: is this a fork, a PR, or a local patch? If flowctl is a dependency, not a fork, these milestones require upstream acceptance, which the spec cannot control. If it is a fork, the user now maintains a fork. Neither path is acknowledged.

**Severity:** `major`

---

## 3. Append content selection rule is stated but not specified

**What:** "Content selection rule: reflect produces a structured `## KB entries` section; flowctl appends only that section."

**Why it's wrong:** The `reflect.md` template (from run 03) uses `## Knowledge base updates written` as the section name — not `## KB entries`. The spec introduces a new section name without updating the template, creating a mismatch between what the reflect node produces and what the append mechanism expects.

**Severity:** `major`

---

## 4. `issue.md` copy step is manual friction in disguise

**What:** "`issue.md` copied to `run:` at start. Issue lives in the target repo, copied to `run:issue.md` at workflow start."

**Why it's wrong:** The spec says "copied at workflow start" but does not say by what. If the user must manually copy `issue.md` to `run:` before invoking `flowctl`, that is manual pre-processing that the spec presents as automatic. Flowctl's `--issue` flag (for GitHub issues) handles one source; a `repo:` input in the first node handles another. A copy step implies a separate setup node or a flowctl pre-run hook — neither is specified.

**Severity:** `minor`

---

## 5. The spec says "sequential is better because test-developer can see real code" — this is a convenience rationalization

**What:** "Test-developer running after developer means test-developer can inspect actual code, which improves integration test accuracy."

**Why it's wrong:** Integration tests that depend on inspecting the developer's implementation rather than the spec are testing implementation, not behavior. The original reason to parallelize was to keep them independent — test-developer should implement from `test-design.md`, not from what the developer happened to produce. The spec reframes a limitation (sequential is all flowctl supports today) as a benefit. This is worth noting because it affects test quality: test-developer looking at `src/` for guidance will write tests that pass the current implementation but may not catch regressions.

**Severity:** `minor`
