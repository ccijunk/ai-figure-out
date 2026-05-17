# 03 — Options

Three different bets on the role set. Each makes a different trade between adversarial separation and workflow weight. No ranking.

---

## Option A — Lean SDLC mirror (4 roles)

**Pitch:** Mirror traditional human SDLC roles. Four agents, each maps to a familiar job title. `meta` drops from the agent set and becomes a human-run retrospective using a template.

**Sketch:**
- **ba** → `clarify.md` — what & why.
- **architect** → `design.md` — system shape, includes a `## Test strategy` section.
- **developer** → `implementation.md` — code + impl notes.
- **qa** → `qa-report.md` — combined static review + dynamic test findings, severity-tagged per item.
- `reflect.md` exists as a human template, not an agent stage.

**Tradeoffs:** Simpler. Faster runs (fewer spawns, fewer artifacts). Mental model matches a real solo dev wearing fewer hats. Most aligned with MetaGPT's 5-role structure. But: QA combines static and dynamic critique into one role; the same mindset attacks both, weakening adversarial separation. Architect carrying test design risks shallow test thinking — design pressure typically wins over test pressure when one role does both.

**Biggest risk:** combined QA pulls punches. Whichever axis (review or test) the agent finds harder, it skimps on. Failures slip through silently.

---

## Option B — User's proposal as-is (7 roles, two novel splits)

**Pitch:** Keep the proposed set. Bet on adversarial separation: distinct roles for static (reviewer) vs dynamic (tester) critique, distinct roles for design and test design, agent-as-meta-role for retrospectives.

**Sketch:**
- **ba** → `clarify.md`
- **architect** → `design.md`
- **test-arch** → `test-design.md` (parallel branch with developer)
- **developer** → `implementation.md`
- **reviewer** → `review.md` (static, code-focused)
- **tester** → `test-report.md` (dynamic, behavior-focused)
- **meta** → `reflect.md` (retrospective, non-blocking)

**Tradeoffs:** Strong adversarial separation. Forces test thinking up front (test-arch fires before developer). Static and dynamic critique come from agents with different access (reviewer sees code + design; tester sees runtime). But: `test-arch` and `meta` are novel splits with no prior art — you bet that splitting them outperforms collapsing them, with no benchmark.

**Biggest risk:** `test-arch` and `tester` produce overlapping artifacts because no crisp rule pins "what test-arch *plans* vs what tester *did*." They drift into writing the same lists with different titles.

---

## Option C — Maximum adversarial pressure (9 roles, conditional designer)

**Pitch:** Maximize critical pressure on the implementation. Add a software-type-aware **designer** node and a separate **security-reviewer** axis so the generalist reviewer isn't asked to be a security specialist too.

**Sketch:**
- **ba** → `clarify.md`
- **architect** → `design.md`
- **test-arch** → `test-design.md`
- **designer** → `ui-design.md` — fires only when `software_type == "ui"` (conditional node)
- **developer** → `implementation.md`
- **reviewer** → `review.md` (correctness, maintainability)
- **security-reviewer** → `security-review.md` (independent adversarial axis)
- **tester** → `test-report.md`
- **meta** → `reflect.md`

**Tradeoffs:** Maximum critical pressure: every dimension (design, security, code-quality, behavior) has its own dedicated adversarial role. The conditional `designer` node sets a precedent for software-type branching the `spec-to-code` spec already requires. But: 9 roles is heavy. Most solo-dev tasks don't justify this much separation — workflows often get abandoned for ad-hoc Claude Code when overhead becomes painful.

**Biggest risk:** workflow becomes too heavy for everyday use. Adoption fails not because the roles are wrong but because spinning up 9 agents for a 50-line PR feels absurd, so the workflow goes unused.

---

## How the options differ (not part of the spec)

| | A | B | C |
|---|---|---|---|
| Agent count | 4 | 7 | 9 |
| Adversarial axes | 1 (qa) | 2 (review, test) | 3 (review, security, test) |
| Test-design as role | folded into architect | distinct | distinct |
| Meta as role | dropped (human template) | included | included |
| Software-type adaptation | none | none | conditional designer |
| Prior art match | MetaGPT-like | partial (novel splits) | none |
| Failure mode | combined-role pulls punches | role overlap (test-arch/tester) | too heavy → abandoned |






## what i want 

In business analysis, we should design business models and domain models that are good. The characteristics of business models are not contradictory; they align with the business lifecycle without conflict. They are complete, and unrelated characteristics are orthogonal—meaning they largely do not affect each other. 

Architects know business and technology , he should chanllege the busisness  without conflict, if so he should focus on how to transform business models into code implementation, preferably using Domain-Driven Design (DDD). As long as the business model is sound, it is considered good. Implementation then considers how to leverage technology to support that business model. 

Test designers and test architects need to design testing approaches that ensure the software remains usable throughout its lifecycle. This involves developing test cases to safeguard the entire software. For every feature addition or bug fix, they should do so accordingly.They also need to perform regression testing and the like, and they need to develop scripts to safeguard the entire software.