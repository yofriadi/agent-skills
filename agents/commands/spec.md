---
description: Start spec-driven development — write a structured specification before writing code
---

Read and strictly follow the `skill://spec-driven-development` skill for this task.

Start by reading the existing discovery artifacts that define the surface area for this work — relevant `docs/research/*.md`, `docs/spikes/*/SPIKE_FINDINGS.md`, and any prior spec if one exists. Treat them as source material, not optional background.

Before asking the human anything, extract and list:

1. Grounded facts already established by research/spikes
2. Assumptions you still need to make to move forward
3. Open questions that materially affect scope, architecture, or verification

Ask only delta questions for unresolved items. Do not restart discovery from scratch when prior phases already answered it.

Then generate a structured spec that:

1. Covers the six core areas: objective, commands, project structure, code style, testing strategy, and boundaries
2. Adds explicit **In Scope** and **Out of Scope** sections when the work is phased, architectural, or intentionally partial
3. Includes concrete **Success Criteria** and **Open Questions**
4. Cites the research/spike artifacts that grounded major scope or design decisions

Save the spec as `SPEC.md` in the project root, stop after the spec, and ask for human approval before `/plan`, `/tasks`, or implementation.

$@
