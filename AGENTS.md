# AGENTS.md

## Task Completion Requirements

- Both `bun lint` and `bun typecheck` must pass before considering tasks completed.
- NEVER run `bun test`. Always use `bun run test` (runs Vitest).

## Project Snapshot

Agents is a minimal web GUI for using code agents — Codex, Gemini, and Claude Code are all supported providers.

This repository is a VERY EARLY WIP. Proposing sweeping changes that improve long-term maintainability is encouraged.

## Core Priorities

1. Performance first.
2. Reliability first.
3. Keep behavior predictable under load and during failures (session restarts, reconnects, partial streams).

If a tradeoff is required, choose correctness and robustness over short-term convenience.

## Maintainability

Long term maintainability is a core priority. If you add new functionality, first check if there are shared logic that can be extracted to a separate module. Duplicate logic across multiple files is a code smell and should be avoided. Don't be afraid to change existing code. Don't take shortcuts by just adding local logic to solve a problem.

## Package Roles

- `apps/server`: Node.js WebSocket server. Manages provider sessions (Codex, Gemini, Claude Code), serves the React web app, and coordinates orchestration domain events.
- `apps/web`: React/Vite UI. Owns session UX, conversation/event rendering, and client-side state. Connects to the server via WebSocket.
- `apps/desktop/electron`: Electron desktop shell. Spawns a desktop-scoped `agents` backend process, exposes the `DesktopBridge` IPC, and loads the shared web app.
- `packages/contracts`: Shared Effect/Schema schemas and TypeScript contracts for provider events, WebSocket protocol, and model/session types. Keep this package schema-only — no runtime logic.
- `packages/shared`: Shared runtime utilities consumed by both server and web. Uses explicit subpath exports (e.g. `@agents/shared/git`) — no barrel index.

## Provider Architecture

Agents supports three providers. Each is implemented as an Effect Layer in `apps/server/src/provider/Layers/`.

### Codex & Gemini

- Long-lived JSON-RPC daemons spawned once per session (via `codex app-server` / `gemini app-server`).
- Session startup/resume and turn lifecycle managed in `codexAppServerManager.ts` / `geminiAppServerManager.ts`.

### Claude Code

- **Subprocess-per-turn**: spawns `claude -p --output-format stream-json` for each turn. Sessions persist via `~/.claude/projects/` (Claude CLI manages this). Session continuity is achieved by passing `--resume <session_id>` on subsequent turns.
- Session lifecycle and state types: `claudeCodeAppServerSession.ts`
- Manager (spawn, NDJSON stream parsing, control protocol): `claudeCodeAppServerManager.ts`
- Pure utilities (tool classification, stderr parsing, control messages): `claudeCodeAppServerHelpers.ts`
- Effect Layer wrapping manager → adapter contract: `provider/Layers/ClaudeCodeAdapter.ts`
- Minimum supported CLI version: **v2.0.0** (validated in `provider/claudeCodeCliVersion.ts`)
- Runtime modes:
  - `full-access`: passes `--dangerously-skip-permissions`, no control protocol
  - `approval-required`: uses lightweight stdin/stdout control protocol for tool approvals

### Provider Registration

All adapters are registered in `serverLayers.ts` via `ProviderAdapterRegistryLive` and routed through `ProviderService`.

## Key Server Files

| File                                             | Role                                                                |
| ------------------------------------------------ | ------------------------------------------------------------------- |
| `src/serverLayers.ts`                            | Composes all Effect Layers; registers all provider adapters         |
| `src/wsServer.ts`                                | WebSocket server; routes JSON-RPC methods to services               |
| `src/claudeCodeAppServerManager.ts`              | Claude Code session manager (per-turn subprocess)                   |
| `src/claudeCodeAppServerSession.ts`              | Claude Code session types & helpers                                 |
| `src/claudeCodeAppServerHelpers.ts`              | Claude Code pure utilities                                          |
| `src/provider/Layers/ClaudeCodeAdapter.ts`       | Claude Code Effect adapter layer                                    |
| `src/provider/Layers/ProviderAdapterRegistry.ts` | In-memory adapter registry                                          |
| `src/provider/Layers/ProviderService.ts`         | Routes API calls to registered adapters                             |
| `src/orchestration/`                             | Domain event engine; projects runtime events → orchestration events |
| `src/checkpointing/`                             | Diff tracking per session turn                                      |
| `src/git/`                                       | Git operations                                                      |
| `src/terminal/`                                  | PTY management (Bun or node-pty)                                    |
| `src/persistence/`                               | SQLite-backed event store & repositories                            |

## Web App

- Framework: React 19 + Vite + TanStack Router
- State: Zustand (`store.ts`) with localStorage persistence
- Session logic: `session-logic.ts`
- WebSocket RPC client: `nativeApi.ts` / `wsNativeApi.ts`
- UI components: Base UI + Tailwind CSS v4 + shadcn/ui (base-mira theme)
- Rich text: Lexical editor for composer, react-markdown for rendering
- Terminal: xterm.js with fit addon
- Provider types: `"codex" | "gemini" | "claude-code"`
- Session phases: `"disconnected" | "connecting" | "ready" | "running"`
- Env vars: `VITE_WS_URL` (custom WebSocket URL), `VITE_NATIVE_API_DISABLED` (desktop webview mode)

## Tooling

| Tool        | Purpose                                          |
| ----------- | ------------------------------------------------ |
| Bun         | Package manager, runtime (required: ≥1.3.9)      |
| Node.js     | Server runtime (required: ^22.13 \|\| ≥24.10)    |
| tsdown      | TypeScript bundler for server/contracts/electron |
| Vite 7      | Web app bundler and dev server                   |
| oxlint      | Linting (Rust-based, replaces ESLint)            |
| oxfmt       | Formatting (Rust-based, replaces Prettier/Biome) |
| Vitest      | Unit and browser tests                           |
| Electron 40 | Desktop shell                                    |

- Default ports: server `3773`, web `5733`
- Set `AGENTS_DEV_INSTANCE=<name>` to shift all ports together for multiple dev instances
- Set `AGENTS_PORT_OFFSET=<n>` for explicit numeric port offset

## WebSocket Protocol

- **Request/Response**: `{ id, method, params }` → `{ id, result }` or `{ id, error }`
- **Push events**: `{ type: "push", channel, data }`
- Key methods: `providers.startSession`, `providers.sendTurn`, `providers.interruptTurn`, `providers.respondToRequest`, `providers.respondToUserInput`, `providers.stopSession`, `shell.openInEditor`, `server.getConfig`, `orchestration.getSnapshot`, `orchestration.dispatchCommand`
- Key push channel: `orchestration.domainEvent` (runtime activity projected into orchestration events server-side)

## Reference Repos

- Open-source Codex repo: https://github.com/openai/codex
- Codex-Monitor (Tauri, feature-complete, strong reference implementation): https://github.com/Dimillian/CodexMonitor

Use these as implementation references when designing protocol handling, UX flows, and operational safeguards.
