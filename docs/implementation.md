# Gemini CLI implementation details

This document is a developer-oriented “code tour” of Gemini CLI. It focuses on
where the important logic lives and how data flows through the system at
runtime.

If you’re looking for the product-level view first, read
[Architecture overview](./architecture.md).

## Repository layout (monorepo)

Gemini CLI is an NPM workspaces monorepo. The primary code lives under
`packages/`.

- `packages/cli`: The user-facing terminal app (Ink/React UI + argument parsing)
- `packages/core`: Shared “backend” library used by the CLI (tools, policies,
  model calls, sessions, MCP, hooks)
- `packages/a2a-server`: Optional HTTP server wrapping core as an A2A agent
- `packages/vscode-ide-companion`: VS Code extension that exposes IDE context +
  diff editing to the CLI via MCP
- `packages/test-utils`: Shared test helpers
- `integration-tests/`: Vitest-based end-to-end style tests for the CLI
- `scripts/`: Build/lint/release automation

## Entrypoints and packaging

### Development entry

`npm run start` runs `scripts/start.js`, which launches the CLI package in
development mode.

### Published binary

The published executable is a bundled file:

- Root `package.json` maps the `gemini` bin to `bundle/gemini.js`.
- `esbuild.config.js` bundles `packages/cli/index.ts` into `bundle/gemini.js`.

The CLI package itself also has a `bin` entry (`packages/cli/package.json`),
which points at `dist/index.js` for local workspace builds.

### CLI entrypoint

The runtime entrypoint is `packages/cli/index.ts`, which calls `main()` in
`packages/cli/src/gemini.tsx` and applies top-level error handling + cleanup.

## Startup flow (CLI)

The boot sequence is split between “pre-UI” startup and “post-render”
initialization.

### 1) Parse args + load settings

`packages/cli/src/gemini.tsx` uses:

- `packages/cli/src/config/settings.ts` for loading/merging settings.
- `packages/cli/src/config/config.ts` for argument parsing (`yargs`) and
  constructing the core `Config`.

`loadCliConfig()` (in `packages/cli/src/config/config.ts`) is the bridge from
CLI settings/flags → core configuration. It sets up things like:

- Model selection and output format
- Tool allow/deny lists and “approval mode” (default/auto_edit/yolo)
- Workspace trust / trusted folders gating
- Memory (GEMINI.md) discovery and import behavior
- MCP servers, extensions, hooks, and policy configuration

### 2) Pre-UI initialization

`packages/cli/src/core/initializer.ts` runs before rendering the React UI:

- Performs initial auth selection/validation
- Validates theme settings
- Optionally connects the IDE client when “IDE mode” is enabled

### 3) Interactive vs non-interactive mode

`packages/cli/src/gemini.tsx` decides between:

- **Interactive**: Renders the Ink app (`packages/cli/src/ui/AppContainer.tsx`)
- **Non-interactive**: Runs a single prompt to completion and exits
  (`packages/cli/src/nonInteractiveCli.ts`)

In interactive mode, `Config.initialize()` is called inside `AppContainer` once
the UI is mounted. In non-interactive mode, `Config.initialize()` is called in
`packages/cli/src/gemini.tsx` before executing the prompt.

## The core `Config` object (packages/core)

The central runtime object is `Config` (`packages/core/src/config/config.ts`).
It’s constructed by the CLI and then initialized once per session.

During `Config.initialize()` the core wires up most subsystems:

- **ToolRegistry**: Registers built-in tools and discovers additional tools
  (`Config.createToolRegistry()`, `packages/core/src/tools/tool-registry.ts`)
- **MCP**: Starts configured MCP servers and discovers their tools/resources
  (`packages/core/src/tools/mcp-client-manager.ts`)
- **Extensions**: Starts active extensions via an `ExtensionLoader`
  (`packages/core/src/utils/extensionLoader.ts`)
- **Hooks**: Initializes the hook system if enabled (`packages/core/src/hooks/`)
- **GeminiClient**: Starts a chat session and installs tool declarations into
  the model client (`packages/core/src/core/client.ts`)

The CLI imports `@google/gemini-cli-core` directly, so “CLI ↔ core” boundaries
are library boundaries (not IPC), but they’re still separated by API shape:
events, tool scheduler contracts, and `Config`/`GeminiClient` methods.

## Cross-cutting events and logging

Most “core → UI” messaging is done through `coreEvents`
(`packages/core/src/utils/events.ts`). It’s a typed `EventEmitter` with a small
in-memory backlog so early startup messages aren’t lost before the UI
subscribes.

Examples of signals delivered via `coreEvents`:

- User-facing feedback (`CoreEvent.UserFeedback`)
- Captured console logs (`CoreEvent.ConsoleLog`)
- Tool/process output (`CoreEvent.Output`)
- Model/fallback state changes (`CoreEvent.ModelChanged`,
  `CoreEvent.FallbackModeChanged`)

In non-interactive mode, `packages/cli/src/gemini.tsx` installs “fallback”
listeners (see `initializeOutputListenersAndFlush()`) so output is still written
to stdout/stderr when no UI listeners are present.

## Conversation lifecycle (prompt → streamed response)

At a high level, each user prompt drives an agentic loop:

1. Input is normalized (slash commands, `@` file imports, etc.)
2. A model request is started and streamed
3. If the model requests tool calls, those calls are validated, confirmed, and
   executed
4. Tool outputs are returned to the model as `functionResponse` parts
5. Streaming continues until the model finishes the turn

### Streaming surface: `ServerGeminiStreamEvent`

The core exposes a streamed event surface (see `packages/core/src/core/turn.ts`)
that includes:

- Incremental content tokens
- Thought summaries (when enabled)
- Tool call requests and tool call results
- Compression events, retry events, and finish metadata

In interactive mode, the UI hook `packages/cli/src/ui/hooks/useGeminiStream.ts`
consumes those events and builds the chat history rendered by Ink.

## Tools: definition, scheduling, confirmation

### Tool definitions

Tools are implemented in `packages/core/src/tools/` and modeled as:

- A **declarative tool** (schema + metadata)
- A per-call **invocation** created after schema validation

Key types live in `packages/core/src/tools/tools.ts` (and related files).

### Tool registry

`Config.createToolRegistry()` registers built-in tools (ls/read/grep/edit/shell,
etc.) and then runs discovery:

- “Discovered tools” can be loaded via configured discovery/call commands
  (see `packages/core/src/tools/tool-registry.ts`)
- MCP-provided tools are discovered by `McpClientManager` and registered into
  the same `ToolRegistry`

### Tool execution scheduler

Tool calls requested by the model are orchestrated by `CoreToolScheduler`
(`packages/core/src/core/coreToolScheduler.ts`). It manages a state machine per
tool call:

- scheduled → validating → awaiting_approval → executing → success/error/cancelled

Interactive UI uses `packages/cli/src/ui/hooks/useReactToolScheduler.ts` as the
React wrapper around `CoreToolScheduler`. Non-interactive flows can execute a
single tool call via `packages/core/src/core/nonInteractiveToolExecutor.ts`.

### Confirmation + policy decisions

Before execution, each invocation can require confirmation via
`ToolInvocation.shouldConfirmExecute()`.

There are two cooperating mechanisms:

1. **Per-tool confirmation UX** (tool-specific “what will happen” prompt)
2. **Policy engine** decisions that can allow/deny/ask-user

Core policy components:

- `packages/core/src/policy/policy-engine.ts`
- `packages/core/src/confirmation-bus/message-bus.ts`

The CLI typically constructs the policy configuration from settings in
`packages/cli/src/config/policy.ts` and passes it into the core `Config`.

## Sessions and persistence

Session artifacts are written under the user’s global Gemini directory (usually
`~/.gemini/`) and a per-project temp directory derived from the workspace path.

Notable persistence points:

- Chat/session recording: `packages/core/src/services/chatRecordingService.ts`
  writes JSON session files under `~/.gemini/tmp/<project_hash>/chats/`.
- Workspace settings and extension state: `packages/core/src/config/storage.ts`
  defines `~/.gemini/` and `.gemini/` (per-workspace) paths.

The interactive UI has additional session utilities in
`packages/cli/src/utils/sessionUtils.ts` and `packages/cli/src/utils/sessions.ts`
for listing/resuming/deleting recorded sessions.

## Extensions and MCP integration

Extensions can contribute:

- MCP servers (tools/resources/prompts)
- Additional context/memory
- Custom commands and configuration

The core-side lifecycle is handled by `ExtensionLoader`
(`packages/core/src/utils/extensionLoader.ts`), while the CLI includes
user-facing extension management in `packages/cli/src/config/`.

MCP client orchestration happens in:

- `packages/core/src/tools/mcp-client-manager.ts`
- `packages/core/src/tools/mcp-client.ts`

## IDE integration

IDE integration is built on MCP:

- The core has an IDE client and context store in `packages/core/src/ide/`.
- The VS Code companion extension (`packages/vscode-ide-companion`) runs a local
  Streamable HTTP MCP server (`packages/vscode-ide-companion/src/ide-server.ts`)
  that publishes IDE context updates and supports diff-based edits.

## Where to start reading code

If you want to understand the end-to-end behavior quickly, these are good entry
points:

- CLI boot + mode selection: `packages/cli/src/gemini.tsx`
- Interactive UI shell: `packages/cli/src/ui/AppContainer.tsx`
- Stream processing + tool call loop (UI): `packages/cli/src/ui/hooks/useGeminiStream.ts`
- Core config + initialization: `packages/core/src/config/config.ts`
- Model client wrapper: `packages/core/src/core/client.ts`
- Streaming turn loop: `packages/core/src/core/turn.ts`
- Tool orchestration: `packages/core/src/core/coreToolScheduler.ts`
- Tool registry + discovery: `packages/core/src/tools/tool-registry.ts`
- Policy and confirmation routing: `packages/core/src/policy/policy-engine.ts`,
  `packages/core/src/confirmation-bus/message-bus.ts`
