# Task 3 prompt: Review an ADR like an architecture board

## Inputs (from the student's message)

- `adr`: path to the ADR to review
- `language` (optional): `Thai` or `English` (default). See *Output language* in `AGENTS.md`.

## Instructions

Act as a critical architecture reviewer on an architecture review board. Review the `adr` in the style of ATAM (Architecture Tradeoff Analysis Method).

You may read the `adr` and `docs/scenario.md`, and nothing else. Do **not** edit the ADR.

Write your review to `exercise/outputs/3-review.md`. Start the file with a line stating your agent and model, if you know them. Then include:

1. **Quality-attribute scenarios.** Two or three concrete scenarios (source · stimulus · environment · response · response measure) that this decision must satisfy. Use numbers from the scenario where they exist.
2. **Findings table** with columns `Type | Finding | Why it matters`, where Type is one of:
   - **Risk**: a decision that may lead to an undesirable outcome
   - **Non-risk**: a decision that is sound, and why
   - **Sensitivity point**: a parameter the quality attribute is highly sensitive to
   - **Trade-off point**: a decision that affects two or more quality attributes in opposite directions
3. **Missing or weak.** Options not considered, consequences not stated, and claims in the ADR that are unsupported or likely wrong.
4. **Verdict.** Accept / accept with changes / reject, with the three most important changes.

Keep the whole review to about one page: it will be read aloud to a review board.

Be specific to this system and this ADR. Avoid generic advice that would apply to any system.
