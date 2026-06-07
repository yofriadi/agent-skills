This file provides guidance to AI coding agents (Antigravity, etc.) when working with code in this repository.

## Repository Overview

A collection of skills for senior software engineers. Skills are packaged instructions and scripts that extend your coding agents' capabilities.

## Orchestration: Personas, Skills, and Commands

This repo has three composable layers. They have different jobs and should not be confused:

- **Skills** (`skills/<name>/SKILL.md`) — workflows with steps and exit criteria. The *how*. Mandatory hops when an intent matches.
- **Personas** (`agents/<role>.md`) — roles with a perspective and an output format. The *who*.
- **Slash commands** (`agents/commands/*.md`) — user-facing entry points. The *when*. The orchestration layer.

Composition rule: **the user (or a slash command) is the orchestrator. Personas do not invoke other personas.** A persona may invoke skills.

The only multi-persona orchestration pattern this repo endorses is **parallel fan-out with a merge step** — used by `/ship` to run `code-reviewer`, `security-auditor`, and `test-engineer` concurrently and synthesize their reports. Do not build a "router" persona that decides which other persona to call; that's the job of slash commands and intent mapping.

See [agents/README.md](agents/README.md) for the decision matrix and [references/orchestration-patterns.md](references/orchestration-patterns.md) for the full pattern catalog.

**Antigravity interop:** the personas in `agents/` work as specialized agent personas (referenced when requesting reviews or audits).

## Repository Scope

This repository is **vendor-neutral**. It targets coding agents as a class (Antigravity, etc.) and avoids coupling to any single host's distribution format.

**Out of scope by design:**

- **Claude Code** — no `.claude-plugin/marketplace.json`, no `agents/commands/<name>.md` mirrored to `.claude/commands/`, no Claude-specific slash command surface. Skills, personas, and commands under `agents/commands/` are the canonical form.
- **Other host-specific marketplaces** (Cursor, Windsurf, Gemini CLI, OpenCode) — no per-host plugin metadata. Any host integration notes live in `docs/` as prose, not as config the host consumes.

**Why:** skills and personas are reused across multiple agent hosts. Hosting them under one vendor's marketplace directory makes the tree misleading for every other host and turns a structural choice into a tribal one. The contents are portable; the directory layout should reflect that.

If a future host needs its own metadata, add it as a sibling (e.g. `.cursor-plugin/`) and keep `agents/` as the source of truth.

## Creating a New Skill

### Directory Structure

```text
```

skills/
  {skill-name}/ # kebab-case directory name
    SKILL.md # Required: skill definition
    scripts/ # Required: executable scripts
      {script-name}.sh    # Bash scripts (preferred)
  {skill-name}.zip        # Required: packaged for distribution

```text

### Naming Conventions

- **Skill directory**: `kebab-case` (e.g. `web-quality`)
- **SKILL.md**: Always uppercase, always this exact filename
- **Scripts**: `kebab-case.sh` (e.g., `deploy.sh`, `fetch-logs.sh`)
- **Zip file**: Must match directory name exactly: `{skill-name}.zip`

### SKILL.md Format

```markdown
---
name: {skill-name}
description: {One sentence describing what the skill does, followed by one or more "Use when" trigger conditions. Include trigger phrases like "Deploy my app" or "Check logs" when helpful.}
---

# {Skill Title}

{Brief overview of what the skill does and why it matters.}

## How It Works

{Numbered list explaining the skill's workflow}

Equivalent headings like `Workflow`, `Core Process`, or `When to Use` are fine when they communicate the same structure clearly.

## Usage (Optional)

Include this section only if the skill ships runnable helpers under `scripts/`. Markdown-only skills can omit both the section and the directory entirely.

```bash
bash skills/{skill-name}/scripts/{script}.sh [args]
```

**Arguments:**

- `arg1` - Description (defaults to X)

**Examples:**
{Show 2-3 common usage patterns}

## Output

{Show example output users will see}

## Present Results to User

{Template for how Claude should format results when presenting to users}

## Troubleshooting

{Common issues and solutions, especially network/permissions errors}

```text

### Best Practices for Context Efficiency

Skills are loaded on-demand — only the skill name and description are loaded at startup. The full `SKILL.md` loads into context only when the agent decides the skill is relevant. To minimize context usage:

- **Keep SKILL.md under 500 lines** — put detailed reference material in separate files
- **Write specific descriptions** — helps the agent know exactly when to activate the skill
- **Use progressive disclosure** — reference supporting files that get read only when needed
- **Prefer scripts over inline code** — script execution doesn't consume context (only output does)
- **File references work one level deep** — link directly from SKILL.md to supporting files

### Script Requirements

- Use `#!/bin/bash` shebang
- Use `set -e` for fail-fast behavior
- Write status messages to stderr: `echo "Message" >&2`
- Write machine-readable output (JSON) to stdout
- Include a cleanup trap for temp files
- Reference the script path as `skills/{skill-name}/scripts/{script}.sh`

### Creating the Zip Package

After creating or updating a skill:

```bash
cd skills
zip -r {skill-name}.zip {skill-name}/
```
