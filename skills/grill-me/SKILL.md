---
name: grill-me
description: Stress-tests a spec, plan, or design against existing code, docs, research, spikes, and prior decisions. Use only when the user explicitly invokes it ("grill me", "grill this", "pressure-test this", "stress-test this design/spec/plan") or asks for a technical challenge session rather than a new spec.
---

# Grill Me

## Overview

Some ideas are clear enough to spec but still too fuzzy to build safely. The danger is no longer "we don't know what you want" — it's "the draft sounds plausible but disagrees with the codebase, existing research, or the real constraints." This skill is the technical pressure-test that happens after initial discovery and before execution.

`interview-me` finds what the user actually wants. `grill-me` challenges whether the current artifact is technically coherent, scoped correctly, and consistent with the repository's existing language and decisions.

## When to Use

Apply this skill when:

- The user explicitly says: **"grill me"**, **"grill this"**, **"pressure-test this"**, **"stress-test this spec"**, **"stress-test this plan"**, or equivalent
- A spec, design note, research summary, or plan already exists and needs technical challenge
- The work is architectural, integration-heavy, migration-heavy, or constrained by prior research/spikes
- You need to verify that terminology, boundaries, and success criteria match the actual codebase and prior decisions

**When NOT to use:**

- The user wants open-ended intent discovery from scratch — use `interview-me`
- There is no artifact, draft, or decision to challenge yet
- The task is mechanical, self-contained, or trivial
- The user did not ask for this style of challenge

## Loading Constraints

This skill needs an interactive user. Ask one question at a time and wait for feedback. Do not run it implicitly inside `/spec`, `/plan`, or implementation work.

## The Process

### Step 1: State what you're grilling

Begin with a tight framing:

```text
ARTIFACT: SPEC.md section "Minimal Coding Agent"
GOAL: Find scope gaps, contradictions with prior research/spikes, and ambiguous technical terms before planning.
CURRENT READ: The phase aims to ship non-interactive `pi -p` with streaming output and compatible tools, while deferring TUI, extensions, and broad parity.
```

If the artifact is not explicit, ask the user to point at the exact spec/plan/design section first.

### Step 2: Read the source of truth before asking

Before asking the user anything:

1. Read the artifact being challenged
2. Read the relevant code, docs, ADRs, research notes, and spikes
3. Extract concrete constraints, terminology, and prior decisions
4. Note any contradictions, overloads, or scope leaks

If a question can be answered from repository evidence, answer it yourself instead of asking the user.

### Step 3: Ask one technical question at a time

Format:

```text
Q: <one focused technical question>
WHY IT MATTERS: <what breaks or expands if this stays ambiguous>
RECOMMENDED ANSWER: <your best current answer, grounded in docs/code/research>
```

Wait for the user's response before continuing.

Good questions are about:

- phase boundaries
- compatibility promises
- terminology collisions
- missing invariants
- edge cases
- contradictions between docs and implementation
- decisions that are cheap to clarify now but expensive to reverse later

### Step 4: Challenge the language, not just the solution

When the user or draft uses fuzzy or overloaded terms, force precision.

Examples:

- "compatible tools" → exact compatible subset, schema shape, or behavioral contract?
- "streaming output" → assistant text only, or text + thinking + tool events?
- "natural stop" → first final assistant message, no pending tool calls, or explicit finish reason?
- "minimal" → smallest useful slice for users, or smallest implementation surface for engineering?

Do not let important terms remain rhetorical.

### Step 5: Stress-test with concrete scenarios

Invent specific scenarios that force the artifact to expose its boundaries.

Examples:

- The model emits thinking, then a tool call, then more assistant text. What must `pi -p` print, and in what order?
- A tool schema is accepted by one provider but not another. Is provider breadth in scope for this phase?
- OAuth login works, but refresh races across concurrent CLI invocations. Is auth persistence in scope for the first slice?
- The CLI exits after a final assistant message, then a provider emits one more stream event. What is the contract?

If the draft cannot answer the scenario cleanly, the spec or plan is not ready.

### Step 6: Call out contradictions immediately

Surface contradictions plainly:

- between the draft and prior research
- between user language and repo terminology
- between documentation and code
- between stated scope and implied acceptance criteria

Do not smooth these over. Resolving them is the point of the session.

### Step 7: Close with a decision-oriented restate

When the open questions are burned down enough, restate the result in a form the user can approve:

```text
Here is the technically tightened version:

- Phase outcome: ...
- In scope: ...
- Out of scope: ...
- Hard constraints: ...
- Terms now fixed: ...
- Remaining open questions: ...

Approve / refine?
```

Do not proceed on a vague "sounds good." Get an explicit yes or a concrete correction.

## Output

The output of this skill is a **technically pressure-tested artifact**:

- a tightened statement of phase scope, or
- a corrected spec/plan section, or
- a short list of unresolved technical decisions that must be settled before planning/implementation

If the user wants persistence, save the result into the relevant spec, plan, or design doc.

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| "The spec or plan looks mostly fine, so I should just proceed." | "Mostly fine" means you're leaving scope gaps or contradictions to be resolved during implementation. This leads to code churn and design misalignment. |
| "Grilling takes too much time, we should just start coding." | Clarifying a scope boundary takes 2 minutes. Rewriting code after integration fails takes hours. Grilling is an investment in speed. |
| "I should ask the user all my questions at once." | Batching questions overwhelms the user and leads to vague, low-quality answers. Ask one question at a time to maintain focus. |
| "The user used a term like 'minimal' so I know what it means." | "Minimal" is highly subjective. Always probe and define the exact limits (e.g., what is out of scope). |
| "I shouldn't suggest a recommended answer so they can choose freely." | Giving options works when the user is choosing between trade-offs. Presenting a recommended answer grounded in existing code/research guides them to a safe default. |

## Red Flags

- Batching multiple technical questions in a single response instead of asking one at a time.
- Proceeding to implementation without resolving clear contradictions between the plan and the repository's codebase or spikes.
- Accepting subjective/unclear terms (like "simple", "clean", "robust") without defining their concrete bounds or limits.
- Skipping the step where you check existing docs/code before asking the user (asking a question that could be answered from repo context).
- Grilling without explaining *why* the ambiguity matters or suggesting a recommended answer.
- Letting a contradiction slide because the draft is "close enough".
- Turning the session into broad product discovery instead of a technical challenge.
## Interaction with Other Skills

- **`interview-me`**: upstream. Use it when the user does not yet know what they want.
- **`spec-driven-development`**: adjacent. Use it to write the spec; use `grill-me` only when the user explicitly wants the draft challenged.
- **`planning-and-task-breakdown`**: downstream. Grill first if the plan would otherwise encode ambiguous scope.
- **`technical-discovery-and-research`**: upstream. Grilling should cite and reuse grounded findings rather than rediscovering them.
- **`technical-spike`**: downstream when the grilling session exposes a feasibility question that docs/code cannot settle.

## Verification

After applying this skill:

- [ ] The exact artifact under review was identified
- [ ] Relevant code/docs/research/spikes were read before asking avoidable questions
- [ ] Questions were asked one at a time
- [ ] Each question explained why the ambiguity matters
- [ ] Each question included a recommended answer grounded in evidence
- [ ] At least one terminology clarification ran for any overloaded or fuzzy term
- [ ] At least one concrete scenario was used to stress-test the artifact's boundaries
- [ ] Contradictions with prior evidence were surfaced explicitly
- [ ] The session ended with a decision-oriented restate and explicit approval or correction
