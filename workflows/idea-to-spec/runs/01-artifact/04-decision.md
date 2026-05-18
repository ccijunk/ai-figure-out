# 04 — Decision

## Chosen approach

**Option B (7 roles), with three role re-scopes.**

Option B's pipeline shape and adversarial structure are kept. The re-scopes change what BA, Architect, and Test-arch focus on — not the pipeline order or artifact names.

---

## Role re-scopes

### BA → domain modeler (not requirements gatherer)

BA produces a **domain model**, not a BRD. Output is evaluated against three criteria:

- **Non-contradictory** — no two business rules conflict within the model.
- **Complete** — covers the full business lifecycle for this problem: creation, operation, edge cases, termination.
- **Orthogonal** — unrelated capabilities are independent; changing one does not force changes in another.

This is DDD ubiquitous-language + domain model work. Narrative requirements summaries do not satisfy the contract.

### Architect → DDD bridge and business challenger

Two responsibilities, in order:

1. **Challenge the domain model.** If `clarify.md` has contradictions or lifecycle gaps, flag them and block the fan-out (`test-arch ∥ developer`) until BA re-runs. This is an adversarial gate, not a rubber stamp.
2. **If the model is sound, transform it into code structure via DDD.** Bounded contexts, aggregates, entities, value objects, domain services. Technology choices serve the business model — they do not drive it.

### Test-arch → lifecycle safeguard designer and architecture challenger

Test-arch has two responsibilities:

1. **Challenge the architect.** If `design.md` is not testable enough — missing isolation boundaries, untestable coupling, no observable outputs — flag it. This is an adversarial gate on `design.md`, symmetric to the architect's gate on `clarify.md`.
2. **Design lifecycle safeguards.** Test cases for the full software. Regression script stubs that must pass on every feature addition or bug fix. The artifact is a living strategy, not a one-shot test plan.

---

## Unchanged from Option B

| Element | Status |
|---|---|
| Pipeline shape | kept: `ba → architect → (test-arch ∥ developer) → reviewer → tester → meta` |
| Artifact names | kept: `clarify.md`, `design.md`, `test-design.md`, `implementation.md`, `review.md`, `test-report.md`, `reflect.md` |
| reviewer (static) / tester (dynamic) split | kept |
| meta inside the workflow, non-blocking | kept |

---

## Open questions for the Spec Writer

1. Does `clarify.md` carry an explicit domain model section (entities, bounded contexts), or is the domain model implicit in the business rules? If BA produces a domain model AND architect transforms it, the BA/architect boundary needs a crisp handoff contract.
2. Does the architect's "challenge" output live inside `design.md` as a `## Domain model review` section, or is it a separate gate artifact that only appears when issues are found?
3. Test-arch owns regression scripts conceptually — but scripts are code. Does `test-design.md` include script stubs (filenames + test IDs), or only the test strategy in prose?
