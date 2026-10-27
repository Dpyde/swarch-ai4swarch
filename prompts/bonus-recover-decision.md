# Bonus task prompt: Recover an undocumented decision

## Inputs (from the student's message)

- `decision`: the decision to reconstruct
- `evidence`: the files you may read
- `language` (optional): `Thai` or `English` (default). See *Output language* in `AGENTS.md`.

## Instructions

Nobody wrote an ADR for the `decision` in Online Boutique.

Read only the `evidence` and the ADR template `docs/adr/template.md`. Reconstruct the decision as an ADR using the template, with status **"Reconstructed"**, and write it to `exercise/outputs/bonus-recovered-adr.md`. Start the file with a line stating your agent and model, if you know them.

At the end of the file, add a section **Claim check**: a table with one row per claim in the ADR (context, options, rationale, consequences):

| # | Claim | Type | Evidence |
|---|---|---|---|

where **Type** is one of:
- **Evidenced**: the evidence files directly show it (give file path and symbol)
- **Inferred**: plausible, but not shown in the evidence files
- **Assumption**: general industry knowledge, not specific to this system

Be honest. A reconstructed rationale is usually mostly *Inferred*, and that is the point of the exercise.
