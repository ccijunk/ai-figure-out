# 03 — Options

Three approaches to handling the two-codebase relationship and the gaps between `flowctl`-today and the custom workflow.

---

## Option A — Work strictly within flowctl's current model (no gaps)

**Pitch:** Only use what `flowctl` already supports. Drop or defer any workflow feature that requires parallel branches, join gates, or automated AI-to-AI retry loops.

**Sketch:**
- Pipeline is purely sequential: BA → Architect → Test-arch → Developer → Test-developer → Test-execution → Final review → Reflect.
- Retry loops replaced by human-approval nodes (`flowctl`'s built-in). When a gate would return BLOCKED, a human node fires instead. Human reads the artifact and manually approves or rejects.
- Knowledge-base updates: Reflect writes its `knowledge-base updates` section to `run:reflect.md`; a human manually copies relevant entries to `workflow:knowledge-base/<role>.md` after the run.
- Opencode invoked with `cwd = repo_dir` for developer and test-developer nodes; all other nodes use default workflow context.

**Tradeoffs:** Zero implementation risk — the tool works today. Removes the fully autonomous adversarial loop: a human is in every retry cycle. Workflow is slower per-run but never blocked on a tooling gap. Loss: the AI-to-AI gate pattern from run 02 is gone; learning loop requires human intervention to persist.

**Biggest risk:** Human gates add friction proportional to the number of retry cycles. A BA that requires 3 re-runs triggers 3 human approvals. The workflow becomes slower than just doing it manually.

---

## Option B — Extend flowctl with two primitives: sequential retry loops + KB file append

**Pitch:** Add only the two missing primitives that make the AI-to-AI gate pattern work, while keeping parallel branches as future work (accepting sequential execution for developer + test-developer).

**Sketch:**
- **AI retry loop primitive:** A node can declare `on_blocked: retry_node: <upstream_node_id>`. When the gate sentinel is BLOCKED, flowctl re-invokes the upstream node with the gate's feedback file as an additional input. Max retries from the YAML config.
- **KB append primitive:** A node can declare `outputs: append: workflow:knowledge-base/ba.md`. Flowctl appends the content rather than overwriting. Reflect uses this to update KB files without human mediation.
- Parallel branches (developer ∥ test-developer): deferred. Developer and test-developer run sequentially.
- Path model unchanged: `run:`, `workflow:`, `repo:` as-is.
- Opencode subprocess: invoked with `cwd = repo_dir` for code-writing nodes.

**Tradeoffs:** Two targeted additions preserve the core adversarial loop without parallel execution complexity. Sequential developer → test-developer means test-developer can reference developer's code, which is arguably better than pure parallelism. Cost: forking or contributing to `flowctl`. The KB append primitive requires a flowctl code change.

**Biggest risk:** The `on_blocked` retry loop requires `flowctl` to parse the BLOCKED sentinel from an output file — meaning the orchestrator does light content inspection. This is "semantic validation" that run 02 explicitly deferred to v2. Risk of sentinel-parsing bugs in the orchestrator.

---

## Option C — Treat flowctl as the runner only; implement gate logic as thin scripts

**Pitch:** Keep flowctl as a dumb sequential runner. Put all gate logic — sentinel checking, retry dispatch, KB appending — in thin Python scripts that flowctl calls as script nodes (not LLM nodes).

**Sketch:**
- Every gate (`architect_domain_gate`, `testability_review`, etc.) is a `script` node, not an LLM node. The script reads the previous node's output, checks the sentinel line, and writes a routing signal to `run:gate-signal.json`.
- Flowctl reads `gate-signal.json` via a conditional transition: `if gate-signal.json.result == "BLOCKED" → retry_node`.
- KB append is a script node that reads `run:reflect.md`, extracts `## Knowledge base updates written`, and appends to `workflow:knowledge-base/<role>.md`.
- Parallel branches: still deferred. Sequential.
- Opencode subprocess: invoked with `cwd = repo_dir` and a `--context-file run:design.md` flag pattern.

**Tradeoffs:** Zero changes to `flowctl` core. Gate logic is testable Python scripts, not embedded in the orchestrator. Scripts can be evolved independently of flowctl. But: adds a script per gate (4 gate scripts + 1 KB append script = 5 scripts), which is the actual complexity that has to be maintained.

**Biggest risk:** Five scripts are a second maintenance surface. If `flowctl` changes how conditional transitions work, the scripts may need updates. Also: script nodes feel like fighting the framework rather than using it.

---

## How the options differ

| | A | B | C |
|---|---|---|---|
| AI-to-AI retry loops | No (human gates) | Yes | Yes (via scripts) |
| KB auto-update | No (manual) | Yes | Yes (via script) |
| Parallel dev tracks | No | No | No |
| Flowctl changes required | None | Yes (2 primitives) | None |
| New maintenance surface | None | flowctl fork | 5 scripts |
| Autonomous run possible | No | Yes | Yes |
| Risk | human friction | sentinel parsing | script/flowctl coupling |
