---
name: technical-spike
description: Performs time-boxed, experimental coding (Spikes) to validate technical feasibility, test APIs, or explore architectural ideas. Use before writing a spec or plan when technical risks or unknowns are high.
---

# Technical Spike

## Overview

A "Spike" is a time-boxed, experimental coding session to validate technical feasibility, test new APIs, or explore complex architectural ideas. Spikes are designed to gather knowledge and reduce risk. **The code written during a spike is throwaway—it must not be committed to the main production branch.**

## When to Use

- Evaluating a new library or tool before deciding to adopt it
- Validating if a specific execution path or integration actually works
- Trying out a risky architectural change to see if it compiles and runs
- Exploring performance bottlenecks with experimental profiling

**When NOT to use:** Routine implementation tasks, standard features, or tasks with no technical unknowns.

## The Gated Workflow

A spike follows a tight, risk-reducing loop:

```text
DEFINE GOAL ──→ EXPERIMENT ──→ RUN & EVALUATE ──→ DOCUMENT & CLEAN UP
     │              │                │                   │
     ▼              ▼                ▼                   ▼
 Timebox and    Draft messy       Execute and        Record findings,
 target risk    sandbox code     observe behavior    discard code
```

### Step 1: Define Goal and Timebox

Define exactly what question the spike needs to answer (e.g., "Can we fetch data from API X and serialize it using Zod?"). Keep the scope minimal and the timebox tight (e.g., 15-30 minutes).

### Step 2: Implement Sandbox Code

Write draft code to test the goal.

- Prioritize speed and learning over code quality.
- Ignore linters, test requirements, or formatting rules.
- Do not make load-bearing changes to existing production files.

### Step 3: Run and Observe

Execute the spike code and capture the output:

- Monitor logs, console errors, and database changes.
- Verify performance or memory usage if that is the goal.

### Step 4: Document Findings and Discard Code

Save your findings (lessons learned, configuration gotchas, successful code snippets) in `SPIKE_FINDINGS.md` in the project root (or under the `docs/` folder as `docs/SPIKE_FINDINGS.md`, or a descriptive name under `docs/spikes/` if conducting multiple spikes).
**Critically: Discard all spike code changes.** Do not commit draft spike code directly to production; rewrite the solution cleanly in the subsequent `/build` phase.

**Spike Template:**

```markdown
# Spike: [Spike Objective]

## Question Answered
[e.g., "Yes, the SDK is compatible with our node runtime."]

## Findings & Learnings
- Config setting X is required, otherwise Y fails.
- Performance overhead is negligible (< 10ms).
- Code snippet that worked:
  ```ts
  // ...
  ```

## Next Steps

- Incorporate configuration X into the `/spec` and `/plan`.
- Re-implement the working snippet cleanly in the `/build` phase.

```text

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| "This spike code is good enough, I'll just merge it" | Spike code is rushed and lacks tests, edge-case handling, and proper architecture. Always rewrite it cleanly. |
| "I don't need a spike, I'll just build it on main" | Building risky features directly on main leads to broken builds, rolled-back commits, and long debugging loops. |
| "A spike is a waste of time" | Finding out a library doesn't work after 15 minutes of spikework saves hours of failed implementation later. |

## Red Flags

- Committing draft spike code to main or leaving unused draft files in the repo.
- The spike takes hours or expands in scope without answering the core question.
- Editing production files directly during the spike without a rollback plan.

## Verification

Before completing a spike, confirm:
- [ ] The core question has been answered (yes or no).
- [ ] Findings and working patterns are documented in `SPIKE_FINDINGS.md` (or in the `docs/` folder).
- [ ] All draft code changes have been discarded or isolated.
