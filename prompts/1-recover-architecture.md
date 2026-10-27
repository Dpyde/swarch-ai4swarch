# Task 1 prompt: Recover the architecture

## Inputs (from the student's message)

- `run`: `A` or `B`
- `evidence`: the files and folders you may read

## Instructions

Recover the **C4 container-level architecture** of the Online Boutique system.

Read **only** the `evidence`.

Write the result to `exercise/outputs/1<run>-architecture.md` (e.g. `exercise/outputs/1A-architecture.md`), with these sections in order:

1. **Agent.** Your product name and model, if you know them.
2. **Files read.** The exact files you opened.
3. **Containers table.** One row per container (service or data store): name · technology/language · responsibility · evidence (file path).
4. **Dependencies table.** One row per runtime dependency: caller → callee · protocol · evidence (file path and the symbol or environment variable). Use the most specific evidence you can find.
5. **Mermaid diagram.** A Mermaid `C4Container` diagram (see *Diagrams* in `AGENTS.md`) that matches the two tables exactly: every container is a `Container`, `ContainerDb` or `ContainerQueue` inside a `System_Boundary`, the end user is a `Person`, external systems are `System_Ext`, and every dependency is a `Rel` labelled with its protocol.
6. **Gaps.** Anything you could not determine from the evidence, and which extra file would settle it.

Do not add containers or dependencies that the evidence doesn't show. If you think one exists but can't prove it, list it under **Gaps**, not in the diagram.
