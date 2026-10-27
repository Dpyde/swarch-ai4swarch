# Task 2 prompt: Draft an ADR for a new requirement

## Inputs (from the student's message)

- `run`: `a` (zero-shot) or `b` (with context)
- `context`: `none`, or the files you may read
- `requirement`: the requirement text
- `language` (optional): `Thai` or `English` (default). See *Output language* in `AGENTS.md`.

## Instructions

Draft an architecture decision record for the `requirement` on Online Boutique.

Apart from the ADR template `docs/adr/template.md`, read **only** the `context` files. If `context` is `none`, read nothing else and write from the requirement alone.

Write the ADR to `exercise/outputs/2<run>-adr.md` (e.g. `exercise/outputs/2a-adr.md`). Start the file with a line stating your agent and model, if you know them. Keep every template heading, set status to **"Proposed"**, and number it 0002.

- If an example ADR or scenario is in your context, follow its style and use its facts (quality goals, constraints, numbers).
- If something you need is not given (a number, a constraint, a team skill), don't invent it. List it under **Open questions** at the end.
