# ai-figure-out

A scratch repo for exploratory work — small experiments, one-off investigations, "let me figure out how X works" sessions. Not a product; not a library.

## How to work here

- Each top-level directory is one investigation. Don't mix unrelated experiments in the same folder.
- Prefer the smallest possible reproduction. If a 30-line script answers the question, that's the deliverable — don't grow it into a framework.
- Default to **not** generating supporting files (README, tests, configs) unless I ask. A single script with clear output is usually enough.
- When the investigation is done, leave a 3–5 line note at the top of the main file: what I was trying to figure out, and what we found. Skip the note if the code is so short it'd outweigh it.

## Before writing code

- If the question is ambiguous, ask one clarifying question before coding — don't guess and build.
- If there are multiple reasonable approaches, name them in 2–3 lines each and let me pick. Don't silently choose.

## Push back on me

I am often wrong. Do not just agree with me.

- **Challenge my opinions.** If I assert something that looks shaky, say so and explain why. "You're probably right" is not a useful answer unless you actually checked.
- **Cite evidence, not vibes.** Claims about how the code, a library, or a tool behaves must be backed by something concrete: a file + line, a doc link, a command you ran, or output you observed. If you don't have evidence, say "I'm guessing" out loud — don't present a guess as a fact.
- If I push back and you still think you're right, hold the position and show the evidence. Don't fold just because I sounded confident.

## Conventions

- Python: `uv run` for scripts, ruff defaults. No `requirements.txt` unless dependencies are non-trivial — inline `# /// script` deps are fine.
- Shell: bash, `set -euo pipefail` at the top.
- No build systems, no CI, no packaging unless I ask.

## Don't

- Don't add abstractions for "future reuse" — these are one-shots.
- Don't add error handling for cases that can't happen in a scratch script.
- Don't commit for me. I'll handle git.
