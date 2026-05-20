# 02 — Context

## Relevant existing code

The only direct precedent in this repo is `workflows/idea-to-spec/agents/01-interviewer.md` … `06-editor.md` — six concrete role/artifact contracts already in use. Worth keeping in mind:

- Each contract has the same shape: **Role** / **Input** / **Output** / **Behavior**.
- Each declares output sections by *name* (Problem / Who has it / Why now / …) — not just "a markdown file." That section-naming is what makes artifacts validatable beyond existence.
- Behavior rules are short, imperative, role-specific (Critic: "do not propose fixes"; Architect: "do not pick a winner").

These six are the closest thing to a working template in this repo. The new role set should follow the same shape.

## Prior art — how other multi-agent frameworks split software-building roles

### MetaGPT (FoundationAgents)

Five roles in fixed sequence: **Product Manager → Architect → Project Manager → Engineer → QA Engineer**. Communication is via **structured documents** (PRD, design doc, task list, code, test report) — not dialogue. ([MetaGPT GitHub](https://github.com/FoundationAgents/MetaGPT), [IBM on MetaGPT](https://www.ibm.com/think/topics/metagpt))

This is the closest analog to your proposed set. Differences:
- MetaGPT has a **Project Manager** role that decomposes the architect's output into tasks and assigns them to engineers. Your set has no such role — the workflow's state machine is your dispatcher.
- MetaGPT has **one** QA role (designs + executes tests). You split this into **test-arch** + **tester**.

### ChatDev

Seven roles modeled on a virtual software company: **CEO, CPO, CTO, Programmer, Reviewer, Tester, Designer**. Communication via dialogue. ([MetaGPT-style team agents review](https://atoms.dev/insights/metagpt-style-software-team-agents-foundations-architecture-applications-and-performance-trends/7e48a158cab643e4b8ea7157286a92f2))

Notable: ChatDev splits **Reviewer** (code) from **Tester** (behavior) — matching your design. Also has a **Designer** role you don't have (relevant for UI software).

### CrewAI

Generic role-playing framework, not coding-specific. Common pattern in software-coding crews: **researcher / writer / reviewer** trio. ([crewAI GitHub](https://github.com/crewaiinc/crewai), [framework comparison](https://gurusup.com/blog/best-multi-agent-frameworks-2026)) Reinforces the reviewer-as-distinct-role pattern.

### AutoGen / Microsoft Agent Framework

Conversational, less role-prescribed. Uses GroupChat with a speaker selector. Not directly relevant to your structured-artifact approach but worth knowing: the AI agent ecosystem in 2026 has three clear leaders — LangGraph, CrewAI, AutoGen. ([2026 framework rankings](https://gurusup.com/blog/best-multi-agent-frameworks-2026))

### Traditional SDLC roles

Mainstream SDLC literature locates:
- **Business Analyst** → produces BRD (Business Requirements Document) and Functional Specification. Bridges stakeholder language to technical spec. ([BA in SDLC](https://www.h2kinfosys.com/blog/business-analysts-role-,in-software-development-life-cycle-sdlc/))
- **Technical Architect** → screen layout, business rules, database layout, architectural plan to physical level. ([Design phase in SDLC](https://www.openxcell.com/blog/design-phase-in-sdlc/))

Your `ba → clarify.md` and `architect → design.md` map cleanly onto these. The naming is lighter but the contracts overlap.

## Constraints discovered

- **Splitting reviewer from tester has precedent (ChatDev) and matches the static-vs-dynamic distinction.** You're not inventing this split.
- **Splitting test-design from test-execution does NOT have direct prior art.** Neither MetaGPT, ChatDev, CrewAI, nor traditional SDLC carves out a "test architect" role distinct from QA. This is a novel split — defensible (forces test thinking before code), but you're on your own for designing it.
- **The `meta` / retrospective role has no direct prior art as an AI agent role.** Retros are a human practice (sprint retrospectives, post-mortems). Adding one as an agent is unusual. Could be valuable (closes the learning loop into per-role memory from `07-spec.md`) or could be noise.
- **No prior art has a `project-manager` / dispatcher agent equivalent in your set.** MetaGPT and ChatDev both have one. Your workflow's state machine plays this role — explicit non-agent dispatcher. Worth documenting.
- **Section-named artifact contracts (not "a markdown file") are the standard.** Both MetaGPT and your own `idea-to-spec` agents declare output sections by name. New templates should do the same.
- **Empirical claim:** structured-document handoffs (MetaGPT) tend to outperform pure dialogue (ChatDev) on complex tasks because each artifact's contract is enforceable. ([MetaGPT paper](https://arxiv.org/pdf/2308.00352)) Reinforces the file-state design you already chose.

## Open unknowns

- **No concrete public BRD / design-doc / test-design templates** with section-by-section markdown skeletons surfaced in the search. Plenty of references to "a BRD covers requirements, scope, acceptance criteria" but no usable template files. The Spec Writer must construct them.
- **Per-role memory across roles** — no prior art on how AI agents in MetaGPT or ChatDev share knowledge across runs. They typically don't; each run is fresh. Your `memory/<role>.md` design is novel here and the templates should leave room for it.
- **Designer role gap.** ChatDev has one; you don't. For UI-heavy projects this is a likely gap. Spec Writer should decide: add it, fold into architect, or scope out.
- **Whether `examples in templates` help or hurt executor performance.** Open question #4 from `01-problem.md` — no empirical evidence found either way. Worth a future experiment, not a v1 blocker.
