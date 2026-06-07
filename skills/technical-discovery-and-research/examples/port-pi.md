# Worked Example: Porting Pi (TypeScript) to Rust

This is a worked instance of `port` research — a concrete, non-trivial port to a target language with a materially different memory and threading model. It demonstrates the full template from `SKILL.md` (Step 4) filled out, and is meant to be read alongside the skill's abstract template.

**Skill:** see `../SKILL.md` for the gated workflow, research-type taxonomy, and the abstract template.

## Grounded Codebase Context (Remote)

Paths below reference the source repository at <https://github.com/earendil-works/pi>, not a local working copy. Use these anchors as the grounding standard for `path:line` references in the actual `RESEARCH.md` deliverable.

- **Repository:** <https://github.com/earendil-works/pi>
- **Primary language:** TypeScript (~94% of codebase)
- **Architecture:** monorepo, 4 packages
  - `pi-ai` — provider-agnostic LLM interface (the provider trait surface)
  - `pi-agent-core` — agent loop, tool use, context management
  - `pi-coding-agent` — CLI / REPL entry point
  - `pi-tui` — terminal UI (Ink-based)
- **Why this is `port` and not `refactor`:** the target language (Rust) has a fundamentally different memory and threading model than TypeScript. Naive line-by-line translation will not compile. The behavioral contract matters: CLI flags, prompts, output format, and streaming behavior must be preserved or explicitly downgraded.

**Core interfaces & entrypoints (via remote search):**

- `pi-ai/src/provider.ts:12` — `Provider` interface (the trait surface to preserve as `pi_ai::Provider` in Rust; defines the LLM provider contract)
- `pi-agent-core/src/agent.ts:45` — `Agent` loop state and executor lifecycle (the core orchestration primitive; source of the `&mut self` chain friction in Rust)
- `pi-coding-agent/src/cli.ts:1` — CLI flag surface and entry point (drives the `clap` derive mapping and the deprecation-shim scope)
- **Database / Schema (SQLite context cache):**
  - Table `conversations` (`id` TEXT PRIMARY KEY, `summary` TEXT, `created_at` INTEGER)
  - Table `messages` (`id` TEXT PRIMARY KEY, `conversation_id` TEXT REFERENCES conversations, `role` TEXT, `content` TEXT)
  - *Data transition mapping:* TypeScript schemas use camelCase keys, whereas Rust structs use snake_case for DB fields. Use `serde(rename_all = "camelCase")` decorators to maintain compatibility with existing export files.
  - *Migration mechanism:* `pi-agent-core` currently handles DB initialization via raw SQL strings. The Rust port will use `rusqlite` + embedded SQL scripts (no heavy ORM migration framework needed) to ensure zero startup latency.

## Scoping Answers (captured up front, per skill's Scoping Questions)

1. **Research type:** `port`
2. **Research target:** full system — 4 packages, ~60k LOC
3. **Scope:** whole repo
4. **NFRs:** startup time < 50ms, memory ceiling < 80MB idle, single static binary, no runtime dependencies
5. **Cross-cutting:** observability (logs, metrics, traces), security (no telemetry out by default), accessibility (TUI contrast and keybinds)
6. **Out of scope:** provider-specific optimizations beyond conformance, GUI/IDE integration, plugin marketplace
7. **Known unknowns:** no mature Rust SDK for several LLM providers; TUI library choice between `ratatui` and `crossterm`; async runtime choice between `tokio` and `async-std`

## Translation Table (excerpt — full table in `RESEARCH.md`)

| Source (npm) | Target (crate) | Maturity | License | Notes |
|---|---|---|---|---|
| `zod@3.22` | `serde` + manual validators | mature | MIT/Apache-2.0 | structural validation differs |
| `@anthropic-ai/sdk@0.30` | `reimplement` against Anthropic OpenAPI spec | n/a | n/a | no first-party Rust SDK |
| `openai@4.x` | `async-openai@0.27` | mature | MIT | feature parity ~85%, watch streaming SSE |
| `commander@12` | `clap@4` (derive) | mature | MIT/Apache-2.0 | idiomatic mapping |
| `ink@5` (TUI) | `ratatui@0.28` | mature | MIT | different event model; full rewrite required |
| `vitest@2` | `cargo-test` + `rstest` | mature | MIT/Apache-2.0 | idiomatic mapping |
| `pino@9` (logging) | `tracing` + `tracing-subscriber` | mature | MIT/Apache-2.0 | idiomatic mapping |

**License check:** all target crates above are MIT/Apache-2.0. **No GPL contamination** in the proposed set. This check is mandatory for every new crate added — the target distribution is proprietary-leaning and a single GPL row would block release. Run `cargo-deny` license policy in CI.

## Thread-Safety Boundary Map (source → target)

The source system is single-threaded JavaScript (Node.js event loop). The target (Rust) is multi-threaded by default and enforces `Send`/`Sync` at compile time. Each Pi package must identify shared mutable state and pick a primitive.

| Source concept | Target primitive | Why this matters |
|---|---|---|
| `async/await` (single-threaded event loop) | `tokio` multi-threaded runtime | `Send`/`Sync` bounds force explicit decisions on every shared value |
| Pass-by-reference of mutable objects | `Arc<Mutex<T>>` or channels | the borrow checker rejects naive translation; need to choose interior mutability vs. message passing |
| Module-level `let` singletons | `OnceLock<T>` or `Arc<T>` injected via context | static mutable state without explicit ownership fails to compile |
| Event emitter pattern | `tokio::sync::mpsc` + `select!` | backpressure and shutdown signaling differ from Node's fire-and-forget |

For each Pi package, the research must list every shared mutable value and pick one of: `Arc<Mutex<T>>`, `Arc<RwLock<T>>`, channels, or actor-style messaging. Default to message passing; reach for locks only at the data-layer boundary.

## Tooling & CI/CD Parity

The port must preserve security verification and development lifecycle speeds:

- **Linting & Formatting:**
  - Source: `eslint-plugin-security` blocks unsafe dynamic imports and dynamic regex evaluation.
  - Target: Map to `cargo clippy` and `cargo-deny` configuration targeting unsafe blocks and dynamic regex dependencies (`regex` crate compilation audits).
- **CI/CD Cache Optimization:**
  - TypeScript tests compile in under 10s via `vitest` without caching.
  - Rust target: Compilation takes ~2 minutes. CI must configure `sccache` caching of the `target/` directory to avoid developer feedback regressions.
- **Artifact Constraints:**
  - Node.js deployment uses a 350MB Docker image.
  - Rust target: Build static musl binary. Package in a `scratch` or `alpine` container, targeting an image size under 30MB with minimal security attack surface.

## Behavioral Contract

User-accepted tier: **Library surface** (identical public API per package; CLI flags and prompts may evolve within a deprecation window).

- Public exports of `pi_ai::Provider` must be preserved 1:1
- Public exports of `pi_agent_core::Agent` must be preserved 1:1
- Streaming behavior must be preserved (chunked tokens, tool calls mid-stream, cancellation)
- Error types must map 1:1 to existing source error categories

Drop-in replacement of the CLI is not required; `pi-coding-agent` flags and prompts may evolve during the port, with a deprecation shim for any renamed flag.

## Phasing Recommendation

| Phase | Target | Stop-early milestone |
|---|---|---|
| 1. Foundation | `pi-ai` — port provider trait surface + conformance tests against 2 providers | ships as a standalone library |
| 2. Core | `pi-agent-core` — agent loop, context, tool use | end-to-end agent runs in a test harness without UI |
| 3. Surface | `pi-coding-agent` — CLI flags and prompts | CLI usable for `ask` / `chat` subcommands |
| 4. UI | `pi-tui` — terminal UI on `ratatui` | full interactive REPL |
| **Stop-early** | **Phase 1 + provider conformance test suite** | demonstrates the architecture works, defers UI risk |

The stop-early milestone is the right place for the user to validate the approach before sinking effort into the agent loop and TUI.

## Risk Register

| Severity | Risk | Mitigation |
|---|---|---|
| High | No mature Rust SDK for Anthropic / Gemini / Bedrock | reimplement against OpenAPI spec; reuse `async-openai` patterns where applicable |
| High | `ratatui` event model diverges from Ink's; rewrite is non-trivial | Phase 0 spike with a minimal TUI to validate feasibility before committing |
| High | Borrow-checker friction in agent loop (`&mut self` chains) | design `Agent` trait with `&mut self` + interior mutability for context (`RefCell` or `Mutex`) |
| Med | Streaming SSE behavior divergence (backpressure, chunking, cancellation) | regression test suite comparing token-by-token output to TypeScript reference |
| Med | License contamination (a transitive dep pulls in GPL/AGPL) | `cargo-deny` license check in CI; review every new dep before adding |
| Med | `tokio` vs. `async-std` choice locks in an ecosystem | Phase 0 spike to validate `tokio` against one provider's full request/response cycle |
| Low | CLI flag rename during port | deprecation shim + migration note in CHANGELOG |
| Low | Logging output format change | keep `tracing` JSON output compatible with existing log shippers |

## Open Questions

- **Provider parity at Phase 1:** do we need 100% feature parity, or only the providers actually used by `pi-coding-agent` today? → hand off to `/spike` with `pi-ai` as scope
- **TUI library choice:** confirm `ratatui` vs. direct `crossterm` usage → hand off to `/spike` with a minimal TUI
- **Async runtime:** `tokio` vs. `async-std`? → Phase 0 spike (1-2 day time-boxed)
- **Plugin system:** the current `pi-coding-agent` has a plugin host; is plugin compat a hard requirement? → user decision, blocks Phase 3

## Handoff

Once the user accepts the stop-early milestone (Phase 1 + provider conformance test suite), the research hands off to:

1. `/spike` for the provider-trait surface (1-2 day time-boxed experiment with `async-openai` and Anthropic reimplementation)
2. Then `/spec` for the full system, citing this `RESEARCH.md` as the discovery input
