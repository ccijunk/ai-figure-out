# idea-to-spec workflow

Adversarial multi-agent pipeline that turns a rough idea into a one-page spec.

## Pipeline

```
raw idea → problem → context → options → [you pick] → draft → critique → final spec
```

## How to run

1. Make a run directory and seed it with the template:

   ```
   mkdir -p runs/my-idea
   cp template/00-idea.md runs/my-idea/
   ```

2. Edit `runs/my-idea/00-idea.md` — write your raw idea in plain language. Don't structure it.

3. In Claude Code, in the repo root, say:

   > Run the idea-to-spec workflow on `workflows/idea-to-spec/runs/my-idea`

   Claude reads each agent prompt in `agents/` and runs them in order, pausing at step 4 (the decision gate) for you to pick an option.

## Agents and contracts

| # | Agent | Reads | Writes |
|---|---|---|---|
| 1 | Interviewer | `00-idea.md` | `01-problem.md` |
| 2 | Researcher | `01` | `02-context.md` |
| 3 | Architect | `01`, `02` | `03-options.md` |
| — | **You** | `03` | `04-decision.md` |
| 4 | Spec Writer | `01`–`04` | `05-spec-draft.md` |
| 5 | Critic | `05` (+ context) | `06-critique.md` |
| 6 | Editor | `05`, `06` | `07-spec.md` |

Each step is a separate Claude agent invocation. State lives entirely in files — you can stop anywhere, hand-edit any artifact, and resume from the next step.

## Why six agents instead of one prompt

Each agent has a mindset that's hard to maintain inside a single context:

- **Interviewer** is curious.
- **Architect** is divergent (must produce options, not pick one).
- **Critic** is hostile (must attack, not propose fixes).
- **Editor** is decisive (must take sides per critique item).

A single agent doing all of these pulls punches when it has to critique its own draft. Separation is the entire point of this workflow being adversarial.
