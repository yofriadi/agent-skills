---
name: technical-discovery-and-research
description: Performs deep codebase exploration, documentation analysis, and constraint/migration mapping. Use before writing a spec or plan to ground understanding.
---

# Technical Discovery and Research

## Overview

Perform deep, structured exploration of the codebase, APIs, and dependencies before drafting a specification. This prevents hallucinated designs and ensures that the proposed solution respects existing architectural patterns, library API contracts, and constraints.

## When to Use

- Starting a new feature in an unfamiliar codebase or module
- Integrating a new third-party library, API, or cloud service
- Making architectural modifications that affect existing components
- Investigating complex legacy systems or refactoring targets
- **Porting** a system to a new language, platform, or paradigm
- **Surveying libraries** for an upcoming technology decision
- **Greenfield** projects needing reference-architecture discovery

**When NOT to use:** Simple fixes, minor layout adjustments, or tasks where all affected files and APIs are already fully understood.

## Research Type Taxonomy

Identify the research type first — it determines which sub-steps matter most:

| Type | Typical signal | Primary emphasis |
|---|---|---|
| `feature` | "Add X to module Y" | Step 1 (locate) + Step 2 (trace impact) |
| `integration` | "Use library Z" | Step 3 (verify contracts) + risk register |
| `modification` | "Restructure Y" | Step 2 (blast radius) + behavioral preservation |
| `refactor` | "Clean up Z" | Step 2 (call graph) + migration surface |
| `port` | "Rewrite X in Y" | Source-system audit + translation table + parity contract (see worked example: [`examples/port-pi.md`](examples/port-pi.md)) |
| `library-survey` | "Pick a library" | Step 3 + ecosystem maturity + license |
| `follow-on` | "Ground the next decision after prior research" | Narrow scope + explicit parent-research linkage + decision-oriented output |
| `greenfield` | "Build X from scratch" | Reference-architecture discovery |

## Scoping Questions

Before searching, confirm with the user:

1. **Research type** — which entry from the taxonomy above applies
2. **Research target** — the feature, library, system, or port under investigation
3. **Scope** — single file, module, package, or whole repo
4. **Non-functional requirements** — performance budget, platform targets, memory ceiling, license constraints, compliance scope
5. **Cross-cutting concerns in play** — security boundaries, observability, accessibility, i18n, multi-tenancy
6. **Out of scope** — what this research will *not* answer (prevent sprawl)
7. **Known unknowns** — what the user already knows is uncertain
8. **Parent research / prior decision** — if this is a follow-on stream, which earlier `RESEARCH.md` or `docs/research/*.md` it extends
9. **Decision this stream must ground** — the concrete next move this research should unblock (for example: "pick the first provider to integrate in the Rust port")

Skip questions that are obviously answered. Capture answers in the deliverable.

## The Gated Workflow

Technical discovery follows a systematic search-and-document process:

```text
LOCATE SYMBOLS ──→ READ & ANALYZE ──→ VERIFY APIS ──→ DOCUMENT FINDINGS
       │                 │                │                 │
       ▼                 ▼                ▼                 ▼
   Map files         Trace call       Check docs/      Create report
   and imports       hierarchies      libraries        with anchors
```

The four steps apply to every research type. The bullets under each step vary by type.

### Step 1: Locate and Map Symbols

Identify the starting point by locating existing code that is structurally or conceptually related to the task. Do not guess what files exist.

- **For existing codebases (all types):** Prefer local search and navigation tools (`search`, `find`, `read`, `ast_grep`, `lsp references`, `lsp definition`, `lsp implementation`) to map relevant files, classes, interfaces, databases, key entry points, and imports. Use `mcp__semble_search` when the repo is large, remote, or you need semantic exploration beyond direct text/symbol lookup.
- **For blank projects or new modules (`greenfield`):** Search external repositories (using `mcp__gh_grep_searchgithub`, `mcp__semble_search`, GitHub code search) to discover similar reference projects, boilerplates, or production-grade analogs. Aim for at least 2-3 reference projects.
- **For `port`:** additionally map
  - Package / module boundaries and which are public-API surface to preserve
  - Language-specific constructs in heavy use (discriminated unions, async/await, decorators, reflection, dynamic imports, macros) — these become the translation input
  - Build, test, lint, and release tooling as research inputs (not "external docs") — identify custom quality checks, security lints, deployment packaging targets, and CI compilation caching requirements
- **For ports of remote repositories (no local clone):** do not scrape the GitHub UI page-by-page. Use the fastest available repo-indexing tool — preferred: `mcp__semble_search` (`repo` = source Git URL), `mcp__gh_grep_searchgithub`, or `mcp__context7` — to query and map the source system. Fallbacks in order: clone the source into a scratch directory under `/tmp`, then `search`/`find` locally.
- **For `integration`:** additionally identify
  - Auth/authz model, env vars, pagination shape, error response shape

### Step 2: Trace Call Hierarchies and Data Flow

Trace how data moves through the targeted system to identify constraints, blast radius, and migration surface.

- **For all types:**
  - List function signatures, parameters, and return types of key components
  - Trace database models, schema relationships, index requirements, and N+1 query patterns
  - Identify downstream and upstream dependencies
- **For `refactor` and `port`:** additionally list
  - Blast radius: every caller of every symbol being changed
  - Migration surface (per the `deprecation-and-migration` skill) — who depends on the old shape
  - Memory ownership, lifetime, and threading-model differences in the target language — these dictate which source idioms translate cleanly
  - Thread-safety boundaries in the source system: shared mutable state, locks, channels, async runtime — these map to the target's concurrency primitives and force explicit ownership decisions
  - **For `modification`, `refactor`, and `port` with persistent state:** additionally map
    - Data transition paths and compatibility (e.g., column type changes, nullability constraints, and backfill requirements for existing records)
    - Database/schema locks and downtime risks (e.g., online DDL needs vs. offline migration, lock contention potential)
    - Rollback safety (whether schema/state updates can be cleanly reversed without data loss)
- **For `modification`:** list the existing tests that must continue to pass

### Step 3: Verify External API Contracts and Documentation

If the task requires integrating a new library or external service, follow the `source-driven-development` skill to identify package versions and fetch official documentation.

- **For all types:**
  - Act as a **librarian**: search GitHub or other public repositories (`gh_grep`, `grep.app`) for concrete, production-grade examples of the target libraries/APIs in use
  - Query official, up-to-date documentation using the recommended source hierarchy
  - Verify exact method signatures, package versions, and required environment variables
- **For `port` and `library-survey`:** additionally build
  - An explicit **translation table**: `existing-dep@version → candidate@version (or "reimplement")` with maturity and license notes for each row
  - Verify the license of every target package; flag rows whose license is incompatible with the target distribution's constraints (e.g., GPL contamination in a proprietary distribution). Make this an explicit check, not an aside
  - Cross-check target-language SDK completeness against the upstream OpenAPI spec for any provider
  - **For `port`:** verify tooling compatibility and custom static analysis rules (e.g., mapping source ESLint security rules to target compiler/clippy policies, and scoping CI compilation caching to prevent compile-time regressions)
- **For `integration`:** verify rate limits, retry semantics, and observability hooks

### Step 4: Compile Discovery Findings

Save the findings in `RESEARCH.md` in the project root, or `docs/research/<topic>.md` when running multiple, parallel, or follow-on research streams. The report must be grounded in observed codebase facts, not assumptions.

If this research extends an earlier stream, keep the earlier document immutable and add a new sibling document that links back to it. Follow-on research is not an append-only dumping ground; it must answer one narrower decision cleanly.

Select the template based on the **Research Type Taxonomy**:

- **Discovery Template (Default)**: For `feature`, simple `modification`, and routine `refactor`. Focuses on quick codebase mapping, blast radius, and identifying unknowns.
- **Comprehensive Template**: For `port`, `integration`, `library-survey`, `follow-on`, `greenfield`, high-risk/stateful `modification`, and large/architectural `refactor`. Focuses on risk mitigation, translation, external contracts, and contract stability.

---

#### Template A: Discovery (Default for feature, simple modification, routine refactor)

```markdown
# Technical Discovery: [Feature/Task Name]

**Research type:** [feature | modification (simple) | refactor (routine)]
**Date:** [ISO date]
**Researcher:** [agent or human]

## 1. Scope & Boundaries
- **In scope:** [what this research will answer]
- **Out of scope:** [what it will not answer]

## 2. Grounded Codebase Context
- **Relevant files & anchors (`path:line`):**
  - `path/to/file.ts:12` (Brief note on responsibility/role)
- **Affected patterns:** [conventions to maintain, e.g. dependency injection, error propagation]

## 3. Blast Radius & Compatibility
- **Findings & data flow:** [what was learned during tracing]
- **Blast radius:** [affected callers, dependencies, or APIs]
- **Behavior to preserve:** [what must continue to work; compatibility constraints]

## 4. Unknowns & Spike Targets
- **Key uncertainties:** [what is still high-risk or unverified]
- **Spike candidates:** [what concrete experiment is needed, if any]

## 5. Handoff Recommendation
- **Next step:** [Proceed to /plan | Proceed to /spike]
```

---

#### Template B: Comprehensive (For port, integration, library-survey, follow-on, greenfield, stateful modification, architectural refactor)

```markdown
# Technical Discovery: [Feature/Task Name]

**Research type:** [integration | port | library-survey | follow-on | greenfield | modification (stateful/high-risk) | refactor (architectural)]
**Date:** [ISO date]
**Researcher:** [agent or human]

## 1. Scope & Boundaries
- **Parent research:** [prior `RESEARCH.md` or `docs/research/*.md`, or `none`]
- **Decision this stream grounds:** [the concrete choice this report is meant to unblock]
- **In scope:** [what this research will answer]
- **Out of scope:** [what it will not answer]
- **Non-functional requirements:** [performance, platform, license, compliance]
- **Cross-cutting concerns in play:** [security, observability, a11y, i18n, multi-tenancy]
## 2. Grounded Codebase Context
- **Relevant files & anchors (`path:line`):**
  - `path/to/file.ts:12`
- **Database / Schema context & transition constraints:** [existing invariants, compatibility constraints, schema locks, rollback limits]
- **Source-system structure (for `port`):** [API surface to preserve, idioms, concurrency model]
- **Reference architectures (for `greenfield`):** [2-3 analogous production projects, boilerplates, or reference implementations consulted]

## 3. External Dependencies & API Contracts (for integration / library-survey / port)
- **Libraries used (pinned):** [packages and versions]
- **Translation table (for `port` / `library-survey`):**
  - `source-pkg@version` -> `target-pkg@version` (maturity, license)
- **Tooling & CI/CD Parity (for `port`):** [lint rules, caching, build constraints]
- **Verified API spec:** [endpoints, signatures, auth, rate limits, env vars]

## 4. Behavioral Contract (for port / modification / refactor)
- **Preservation contract:** [Drop-in | Library surface | Protocol only | Incremental]
- **Preserved invariants:** [concrete guarantees, API contracts, error boundaries]

## 5. Risk Register (for port, integration, high-risk modification/refactor)
| Severity | Risk | Mitigation |
|---|---|---|
| [High/Med/Low] | [Description] | [Mitigation strategy] |

## 6. Suggested Implementation Order / Stop-Early Milestone (for non-trivial effort)
- **Suggested starting order:** [smallest foundational piece first; stop-early milestone at the point where a useful subset ships end-to-end with a conformance test suite]

## 7. Next Steps & Open Questions
- **Spike targets:** [experimental sandbox candidates to resolve uncertainties]
- **Open questions:** [unresolved architectural or product decisions for /plan]
```

### Next Steps

1. **Technical Spike (`/spike`):** If there are high technical risks, integrations with unknown behavior, or architectural uncertainties documented in Step 4, proceed to `/spike` to experiment with sandbox code before planning.
2. **Implementation Plan (`/plan`):** Once the technical surface is understood and verified, proceed directly to the planning phase using `/plan`.
   - *Compatibility note: If a specific repository enforces a spec phase between research and planning, hand off to `/spec` at this point.*

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| "I know the codebase well, I don't need to search" | Even in familiar systems, files change. Doing a quick search prevents stale assumptions. |
| "I'll research as I write the code" | Researching mid-coding leads to context switching, messy architecture, and frequent rewrites. |
| "The user explained how to do it" | The user describes the business goal, not the exact imports, dependencies, and code structure. |
| "We can add the risk register later" | Risks identified mid-implementation force rework. Surface them before committing. |
| "All research is the same" | A port requires a translation table and parity contract; an integration needs env-var verification; a greenfield needs reference projects. The shape differs. |

## Red Flags

- Suggesting new dependencies without checking if existing packages already solve the problem.
- Proposing helper functions that already exist in the codebase.
- Designing API calls without checking official SDK signatures.
- Assuming files exist or work a certain way without reading them first.
- Skipping the parity contract on a port (you'll discover the contract during user testing — too late).
- Surveying libraries without checking license compatibility with the target distribution.
- Adding a target-language crate to a translation table without checking its license against the target distribution's constraints (a single GPL row can block release).
- Treating the source system as a black box on a port — every idiom becomes a hidden translation decision.

## Verification

Before declaring discovery complete, confirm:

### 1. General Gates (All research types)

- [ ] Research type identified and scope documented
- [ ] At least 3 relevant codebase files, reference projects, or documentation sources read and cited
- [ ] Findings grounded in `path:line` anchors, cited external documentation, or concrete reference project analysis (as applicable)
- [ ] Findings saved to `RESEARCH.md` or `docs/research/<topic>.md`
- [ ] Recommended next step (`/spike` or `/plan`) explicitly stated

### 2. Integration, Survey, and Follow-On Gates (For integrations / library surveys / follow-on streams)

- [ ] External API structures, environment variables, authentication, and rate limits verified when relevant to the decision
- [ ] Pinned library versions identified when the stream introduces dependency candidates; concrete usage examples from public code identified and referenced when available
- [ ] Parent research and the specific decision this stream grounds are explicitly linked when this is a follow-on stream
- [ ] Every dependency target license checked against target distribution constraints (no license contamination) — only when the stream introduces or replaces dependency candidates

### 3. Port & High-Risk Change Gates (For ports, stateful modifications, large refactors)

- [ ] Behavioral-preservation contract documented (definition of preserved behavior and invariants)
- [ ] Translation table covers every direct dependency and tools/CI parity (for `port`)
- [ ] Source system thread-safety boundaries mapped to target concurrency primitives (for `port`)
- [ ] Suggested implementation order / stop-early milestone included
- [ ] Risk register with severity ratings present

### 4. High-Risk Integration Gates (For integrations with external services or libraries)

- [ ] Risk register with severity ratings present

### 5. Greenfield Gates (For greenfield projects)

- [ ] At least 2-3 reference projects/architectures documented
- [ ] Chosen patterns are grounded in observed production usage

## Examples

For a worked instance of `port` research showing the full template filled out — including translation table, thread-safety boundary mapping, license-contamination check, behavioral contract, phasing, and risk register — see `examples/port-pi.md`. For a worked follow-on stream that narrows a prior port into a first-provider integration decision, see `examples/follow-on-codex-oauth.md`.
