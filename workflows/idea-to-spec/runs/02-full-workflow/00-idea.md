# Idea

Produce a full, executable spec for the `spec-to-code` workflow — superseding runs 00 and 01.

Requirements from the user:
- Every node has: role, inputs, outputs, artifact format (named sections), cycle/retry logic.
- Adversarial gates at every major handoff, not just BA→Architect.
- Parallel tracks after test-arch: developer (TDD) and test-developer run concurrently.
- Knowledge base (role prompts + memory) is consumed by all roles and updated by Reflect.
- Design is a trade-off outcome — the spec must state what is imperfect and why it was chosen anyway.

## Workflow graph (user-provided)

```mermaid
graph TD
    Issue((Issue)) --> BA[BA: Domain Modeling]

    KnowledgeBase[(Role Prompts & Memory)] -.->|used by| BA
    KnowledgeBase -.->|used by| Architect
    KnowledgeBase -.->|used by| TestArch
    KnowledgeBase -.->|used by| Dev
    KnowledgeBase -.->|used by| TestDev

    BA -- "produces domain model (non-contradictory, complete, orthogonal)" --> Gate1

    Gate1{Architect: Model sound?}
    Gate1 -- No --> BA
    Gate1 -- Yes --> Architect

    Architect[Architect: DDD Transformation]
    Architect -- "output: design.md" --> Gate2

    Gate2{Test-architect: Design testable?}
    Gate2 -- No --> Architect
    Gate2 -- Yes --> TestArch

    TestArch[Test-architect: Design Lifecycle Safeguards]
    TestArch -- "output: test strategy" --> Fork(( ))

    Fork --> Dev[Developer: TDD]
    Dev -- "output: code + unit tests" --> ArchReview[Architect: DDD Review]
    ArchReview --> Gate3{Review pass?}
    Gate3 -- No --> Dev

    Fork --> TestDev[Test Developer: Implement tests]
    TestDev --> TestExec[Test Execution]
    TestExec -- "output: test results" --> TestArchReview[Test-architect: Review]
    TestArchReview --> Gate4{Review pass?}
    Gate4 -- No --> TestDev

    Gate3 -- Yes --> Review[Final Review]
    Gate4 -- Yes --> Review

    Review --> Reflect[Reflect / Retrospective]
    Reflect -->|"updates prompts & memory"| KnowledgeBase
```
