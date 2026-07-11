# Pi Monorepo - Architecture Overview

High-level review of the `pi` agent harness monorepo. Scope: whole repo, architecture only.

## 1. What this repo is

Pi is a self-extensible coding agent harness. It ships as an interactive CLI (`pi`) built on top of a layered stack of packages. The design philosophy (see `CONTRIBUTING.md`) is a **minimal, extensible core**: functionality that does not belong in the core is expected to live in extensions.

- Language: TypeScript, ESM only (`"type": "module"`), Node `>=22.19.0`.
- Build/typecheck: `tsgo` (`@typescript/native-preview`), lint/format: Biome.
- Native-strippable TS only in checked sources (no `enum`, `namespace`, parameter properties, etc.).
- Lockstep versioning: all packages share one version (currently `0.80.6`); the root is `0.0.3` and private.

## 2. Package layout and dependency direction

```
pi-ai  (no internal deps)
  ^
  |
pi-agent-core  --> pi-ai
  ^
  |
pi-coding-agent  --> pi-agent-core, pi-ai, pi-tui
  ^
  |
pi-orchestrator  --> pi-coding-agent

pi-tui  (no internal deps)
```

| Package | LOC (src) | Files | Role |
|---|---|---|---|
| `@earendil-works/pi-ai` | ~37k | 149 | Unified multi-provider LLM API |
| `@earendil-works/pi-agent-core` | ~8k | 25 | Provider-agnostic agent runtime + harness |
| `@earendil-works/pi-coding-agent` | ~53k | 163 | The `pi` CLI (interactive/print/rpc modes, tools, extensions) |
| `@earendil-works/pi-tui` | ~12k | 28 | Terminal UI library (differential rendering) |
| `@earendil-works/pi-orchestrator` | ~2k | 13 | Experimental multi-session orchestrator |

Dependencies flow strictly upward; there are no cycles between packages. `pi-tui` is fully independent of the AI/agent stack, which keeps rendering concerns separated from agent logic.

## 3. Layer-by-layer

### 3.1 pi-ai - Unified LLM API

The lowest layer. Normalizes many providers behind one streaming API.

- **Core types** (`types.ts`, ~730 lines): `Model`, `Context`, `StreamOptions`, `Api`, `ProviderId`, `ThinkingLevel`, `Transport` (`sse` | `websocket` | `websocket-cached` | `auto`), and the `AssistantMessageEventStream` abstraction.
- **`api/`**: one module per wire protocol (`anthropic-messages`, `openai-completions`, `openai-responses`, `openai-codex-responses`, `azure-openai-responses`, `google-generative-ai`, `google-vertex`, `mistral-conversations`, `bedrock-converse-stream`). Each has a tiny `.lazy.ts` wrapper so protocol implementations load on demand.
- **`providers/`**: one module per provider (70+), each pairing a factory (`*.ts`) with a generated model catalog (`*.models.ts`). Catalogs are code-generated (`scripts/generate-models.ts` -> `models.generated.ts`); the guidance is to never hand-edit generated files.
- **`auth/`**: credential store, auth context, OAuth (`utils/oauth`). Supports API keys and OAuth device flows.
- **`compat.ts`**: an explicitly temporary shim preserving the old global API (`stream`/`streamSimple`/`complete`, api-registry, env key injection, static catalog reads). New code is meant to use `createModels()` + provider factories; `compat` is slated for deletion once `coding-agent`'s ModelManager migrates.
- **`faux.ts`**: an in-process fake provider used for deterministic tests (the harness test suite depends on this).

Notable design choices:
- Index (`index.ts`) is deliberately **side-effect free** and re-exports only core types; provider factories, API impls, and OAuth are behind subpath exports (`/providers/*`, `/api/*`, `/oauth`, `/compat`). This keeps tree-shaking and cold-start cost low.
- Direct external deps are pinned exactly; provider SDKs (`@anthropic-ai/sdk`, `openai`, `@google/genai`, `@aws-sdk/client-bedrock-runtime`, `@mistralai/mistralai`) are heavy and are the main dependency surface.

### 3.2 pi-agent-core - Agent runtime

Provider-agnostic engine that turns an LLM stream into a stateful agentic loop.

- **`agent.ts`** - `Agent` class: owns transcript state, emits lifecycle events (`message_start/update/end`, `tool_execution_start/end`, `turn_end`, `agent_end`), and exposes **steering** and **follow-up** message queues (`steer()` / `followUp()`, each with `one-at-a-time` or `all` drain modes). A single `activeRun` guard prevents concurrent prompts. State is exposed through a mutable-but-copy-on-assign wrapper.
- **`agent-loop.ts`** - the pure loop (`runAgentLoop` / `runAgentLoopContinue`): calls the stream function, dispatches tool calls (parallel by default via `toolExecution`), and applies `beforeToolCall`/`afterToolCall`/`prepareNextTurn` hooks.
- **`harness/`** - higher-level building blocks reused by the CLI:
  - `agent-harness.ts` (~36k): the full harness wiring.
  - `session/`: a **branching session tree**. Sessions are ordered entries (`message`, `model_change`, `thinking_level_change`, `active_tools_change`, `compaction`, `branch_summary`, `label`, ...). Storage backends: JSONL (`jsonl-repo`/`jsonl-storage`) and in-memory. Context for the model is derived by walking a path through the tree and applying transforms.
  - `compaction/`: context-window management (`shouldCompact`, `compact`, summary generation, branch summarization, token estimation).
  - `skills.ts`, `prompt-templates.ts`, `system-prompt.ts`: resource injection into the system prompt (agentskills.io-style skill blocks).

The clean split here is the strongest architectural feature: `Agent` is transport/provider-agnostic, and the harness adds sessions/compaction/skills without the CLI needing to know loop internals.

### 3.3 pi-tui - Terminal UI

Independent rendering library with differential (diff-based) redraw.

- **`tui.ts`** (~58k): core `TUI`, `Container`, `Component`, overlay system, focus management.
- **`keys.ts`** (~45k) + `keybindings.ts`: extensive keyboard handling including the Kitty keyboard protocol, key release/repeat, and a configurable keybinding manager (`TUI_KEYBINDINGS`).
- **`components/`**: `Editor`, `Markdown`, `SelectList`, `SettingsList`, `Image`, `Loader`, etc.
- **`terminal-image.ts`**: image rendering across Kitty / iTerm2 protocols with dimension parsing for png/jpeg/gif/webp.
- `utils.ts`: width-aware text handling (`visibleWidth`, `truncateToWidth`, ANSI-aware wrapping) using `get-east-asian-width`.

The keybinding guidance in `AGENTS.md` (never hardcode key checks; add to default keybinding maps) is enforced by this layer's design.

### 3.4 pi-coding-agent - The CLI

The largest and most complex package; the actual `pi` binary. Composed of:

- **Entry**: `cli.ts` -> `main.ts` (~30k) parses args and dispatches to a mode.
- **Modes** (`modes/`):
  - `interactive/interactive-mode.ts` (~204k, single file): the full interactive TUI experience. This is by far the largest file in the repo.
  - `print-mode.ts`: non-interactive single-shot output.
  - `rpc/`: `rpc-mode`, `rpc-client`, `rpc-types` - a JSON-RPC control surface (used by the orchestrator and external drivers). `rpc-entry.ts` is a separate bin export.
- **Core** (`core/`):
  - `agent-session.ts` (~108k): the central session object binding an `Agent`, tools, extensions, model resolution, and events. Another very large single file.
  - `agent-session-runtime.ts` / `agent-session-services.ts`: factory/service decomposition around `AgentSession`.
  - `model-registry.ts` (~36k) + `model-resolver.ts` (~24k): model discovery, aliasing, and selection.
  - `session-manager.ts` (~50k), `settings-manager.ts` (~39k): persistence and config.
  - `package-manager.ts` (~82k) + `package-manager-cli.ts` (~26k): the extension/skill package manager.
  - `tools/`: the built-in tools - `read`, `bash`, `edit`, `write`, `grep`, `find`, `ls` - each exposing a `create*Tool` and `create*ToolDefinition`, plus curated bundles (`createCodingTools`, `createReadOnlyTools`). `file-mutation-queue.ts` serializes concurrent file writes.
  - `extensions/`: the extension system (`loader.ts`, `runner.ts`, `types.ts` ~58k). Extensions hook a rich set of lifecycle events (agent start/end, tool call/result, message render, session start/compact/fork/switch, provider request/headers, input, project trust) and can register tools, commands, providers, editors, autocomplete, and UI widgets.
  - `export-html/`: renders a session to a standalone HTML transcript.
- **Cross-cutting**: `config.ts`, `migrations.ts` (session/config schema migrations), `trust-manager.ts` / `project-trust.ts` (workspace trust gating).

### 3.5 pi-orchestrator - Experimental

Marked experimental and may be removed. Supervises multiple `pi` sessions:
- `supervisor.ts`, `rpc-process.ts`: spawn/manage child `pi` processes over the coding-agent RPC surface.
- `ipc/`: client/server/protocol for inter-process messaging.
- `radius.ts`, `serve.ts`, `storage.ts`: scheduling/serving/persistence.

## 4. End-to-end data flow (interactive prompt)

```
user keystrokes
  -> pi-tui (Editor/keys) captures input
  -> interactive-mode builds a user AgentMessage
  -> AgentSession.prompt() (coding-agent core)
       - resolves model (model-registry/-resolver)
       - assembles system prompt (skills, templates, project context)
       - runs extension beforeAgentStart hooks
  -> Agent.prompt() (agent-core)
       -> runAgentLoop -> streamSimple() (pi-ai compat)
            -> provider api/ module -> provider SDK -> LLM
       <- AssistantMessageEventStream events
       -> tool calls dispatched to coding-agent tools (read/bash/edit/...)
       -> beforeToolCall / afterToolCall extension hooks
  <- lifecycle events bubble up
  -> AgentSession appends entries to the session tree (JSONL)
  -> interactive-mode re-renders via pi-tui differential redraw
```

## 5. Observations (architecture-level)

Strengths:
- **Clean layering with upward-only dependencies** and a UI library fully decoupled from agent logic.
- **Extensibility is a first-class design axis**: broad, well-typed extension event surface and pluggable providers/tools/editors, matching the stated "minimal core" philosophy.
- **Provider abstraction is well-factored**: per-protocol `api/` vs per-provider `providers/`, lazy-loaded, side-effect-free index, generated catalogs.
- **Session model is sophisticated**: a branching tree with compaction and derived context is more capable than a flat message list, enabling forking and summarization.
- **Supply-chain discipline**: exact-pinned deps, `min-release-age`, shrinkwrap with lifecycle-script allowlist, lockfile commit gating.

Risks / areas to watch:
- **A few extremely large single files** concentrate complexity: `interactive-mode.ts` (~204k), `agent-session.ts` (~108k), `package-manager.ts` (~82k), and several 35-50k files in `core/`. These are likely change-magnets and hard to review/test in isolation; candidates for decomposition.
- **`compat.ts` is explicitly transitional** and holds a global mutable API-provider registry plus deprecated static catalog reads. The migration to `createModels()`/ModelManager is a known, still-pending cleanup; the global registry is a shared-state hazard for tests/extensions.
- **Generated model catalogs are large and numerous** (e.g. `openrouter.models.ts` ~130k, `vercel-ai-gateway.models.ts` ~80k). They inflate repo size and must only be regenerated via scripts, never hand-edited.
- **Orchestrator is experimental** and depends on the RPC surface; its stability contract with `coding-agent` is not yet firm.
- **No built-in permission/sandbox model** (documented in the README): the agent runs with the launching user's full privileges, and isolation is delegated to containerization patterns. This is an intentional trade-off worth keeping visible.

## 6. Suggested next steps (if a deeper pass is wanted)

- Decompose `interactive-mode.ts` and `agent-session.ts` along clear seams (event handling vs rendering vs session mutation).
- Track and schedule the `compat.ts` -> ModelManager migration to remove the global registry.
- A focused code-quality or security review of `core/tools/bash.ts`, `bash-executor.ts`, and `trust-manager.ts` given they gate command execution.
