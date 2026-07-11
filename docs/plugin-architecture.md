# Pi Plugin (Extension) Architecture

How Pi's plugin system works. In Pi, "plugins" are called **extensions** and live in `packages/coding-agent/src/core/extensions/`.

## 1. What an extension is

An extension is a TypeScript/JavaScript module whose default export is a **factory function**:

```ts
import type { ExtensionFactory } from "@earendil-works/pi-coding-agent";

export default (pi) => {
  pi.on("tool_call", (event, ctx) => { /* ... */ });
  pi.registerTool({ name: "my_tool", /* ... */ });
  pi.registerCommand("hello", { handler: async (args, ctx) => {} });
};
```

`pi` is the `ExtensionAPI`. The factory runs once at load time and does nothing but *register* things (event handlers, tools, commands, shortcuts, flags, renderers, providers). Everything after that is event-driven.

## 2. Extension points

The extension API is the umbrella, but it exposes several distinct extension points:

- **Event handlers** (`pi.on(...)`) - subscribe to ~30 lifecycle events.
- **Tools** (`pi.registerTool`) - new LLM-callable tools with TypeBox parameter schemas, execution, and custom TUI rendering.
- **Commands / shortcuts / flags** - slash commands, keybindings, CLI flags.
- **Providers** (`pi.registerProvider`) - new LLM providers/models, optionally with a custom `streamSimple` handler and OAuth login flow.
- **Renderers** (`registerMessageRenderer` / `registerEntryRenderer`) and **UI** (custom editor, footer, header, widgets, autocomplete).
- **Resources** - skills, prompt templates, and themes, contributed via the `resources_discover` event or a `package.json` `pi` manifest.

## 3. Discovery and loading (`loader.ts`)

`discoverAndLoadExtensions()` collects extension paths from three sources, in order:

1. Project-local: `<cwd>/.pi/extensions/`
2. Global: `<agentDir>/extensions/`
3. Explicitly configured paths (config/CLI)

Within a directory, discovery is one level deep and recognizes:
- a bare `*.ts` / `*.js` file,
- a subdir with `index.ts`/`index.js`,
- a subdir with `package.json` declaring a `pi.extensions` array (the manifest form for multi-file packages; the manifest can also declare `skills`, `prompts`, `themes`).

Modules are imported with **jiti** so extensions can be plain TypeScript with no build step. Two resolution modes handle Pi's own packages (`pi-ai`, `pi-agent-core`, `pi-tui`, `pi-coding-agent`, `typebox`):
- **Node/dev**: jiti `alias` pointing at workspace `dist` or node_modules.
- **Bun compiled binary**: `virtualModules` where those packages are statically bundled into the binary (hence the static `_bundled*` imports at the top of `loader.ts`).

Extensions resolve `@earendil-works/pi-ai` to the **compat** entrypoint, so the old global API keeps working until `compat.ts` is removed. The package manager (`core/package-manager.ts`) installs extension packages into these directories.

## 4. Two-phase lifecycle: registration then binding

There is a **shared `ExtensionRuntime`** object:

- At load time it has **throwing stubs** for all "action" methods (`sendMessage`, `setModel`, `appendEntry`, etc.). Calling them during factory execution throws "runtime not initialized" - registration is allowed, actions are not.
- The `ExtensionAPI` given to each extension writes registrations into that extension's own `Extension` record (maps of handlers, tools, commands, ...) and delegates actions to the shared runtime.
- Later, the active mode calls `ExtensionRunner.bindCore(actions, contextActions, providerActions)`, which **replaces the stubs** with real implementations wired to the live `AgentSession`. `bindCommandContext()` and `setUIContext()` wire in command-only capabilities (fork/newSession/switchSession/reload) and the mode-specific UI.

`registerProvider` is special-cased: calls made during loading are **queued** (`pendingProviderRegistrations`) and flushed in `bindCore` once the `ModelRegistry` exists; after binding, calls take effect immediately with no `/reload`.

## 5. Event dispatch and mutation semantics (`runner.ts`)

`ExtensionRunner` owns all extensions and dispatches events. Most events go through the generic `emit()`, but several have dedicated methods because they let extensions **influence** the pipeline rather than just observe:

- `emitContext` - handlers can rewrite the message list before each LLM call (chained: each sees the previous output).
- `emitToolCall` - can mutate `event.input` in place to patch tool args, or return `{ block: true }` to veto execution (first block wins).
- `emitToolResult` - can replace tool result `content`/`details`/`isError`.
- `emitMessageEnd` - can replace a finalized message (must keep the same role).
- `emitBeforeAgentStart` - can inject a message and/or replace the system prompt (chained across extensions).
- `emitBeforeProviderRequest` / `emitBeforeProviderHeaders` - rewrite the raw provider payload / mutate headers.
- `emitInput` - `transform` chains, `handled` short-circuits.
- `session_before_*` events - return `{ cancel: true }` to abort a compact/fork/switch/tree-navigation.
- `project_trust` - first `yes`/`no` decision wins, `undecided` falls through.

Every handler call is wrapped in try/catch and routed to `emitError` listeners, so **one misbehaving extension cannot crash the run** (except `emitToolCall`, which intentionally does not swallow so a blocking handler's errors surface).

## 6. Contexts and modes

Handlers receive an `ExtensionContext` created lazily via getters (`createContext()`), so values like the current model or signal are read at call time. There are progressively more powerful variants:

- `ExtensionContext` - base: `ui`, `sessionManager` (read-only), `model`, `abort`, `compact`, `getSystemPrompt`, etc.
- `ExtensionCommandContext` - adds session control (`newSession`, `fork`, `navigateTree`, `switchSession`, `reload`) - only safe from user-initiated commands.
- `ReplacedSessionContext` - passed to `withSession()` callbacks after a session is replaced.

The UI is abstracted behind `ExtensionUIContext`; each mode (interactive/rpc/print) supplies its own implementation, and a `noOpUIContext` is used when no UI is available (so extensions can call UI methods safely in print mode).

A **stale-guard** mechanism (`invalidate()` / `assertActive()`) throws if an extension holds onto a `ctx` after the session was replaced or reloaded, forcing correct use of the `withSession` callbacks.

## 7. Conflict resolution

- Tools and commands: first registration per name wins; duplicate command names get `:N` invocation suffixes.
- Shortcuts: a reserved list of core keybindings cannot be overridden (`RESERVED_KEYBINDINGS_FOR_EXTENSION_CONFLICTS`); other conflicts emit a diagnostic and last-wins.

## 8. Flow summary

```
discover paths -> jiti import -> factory(pi) registers into Extension record
   (runtime actions are throwing stubs here)
        |
   mode calls runner.bindCore / bindCommandContext / setUIContext
        |
   runtime stubs replaced with live AgentSession-backed implementations
        |
   AgentSession emits events -> ExtensionRunner.emit*(...) -> handlers
        (handlers observe, transform, block, or take actions via pi.*)
```

The design goal (per `CONTRIBUTING.md`) is a minimal core with a broad, well-typed, mutation-capable extension surface, error-isolated per handler, so most features can live outside the core.
