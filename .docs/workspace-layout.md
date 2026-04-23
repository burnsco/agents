# Workspace layout

- `/apps/server`: Node.js WebSocket server. Manages provider sessions (Codex, Gemini, Claude Code), serves the built web app, and opens the browser on start.
- `/apps/web`: React + Vite UI. Session control, conversation, and provider event rendering. Connects to the server via WebSocket.
- `/apps/desktop/electron`: Electron desktop shell. Spawns a desktop-scoped `agents` backend process, exposes the `DesktopBridge`, and loads the shared web app.
- `/packages/contracts`: Shared Zod schemas and TypeScript contracts for provider events, WebSocket protocol, and model/session types. Schema-only — no runtime logic.
- `/packages/shared`: Shared runtime utilities with explicit subpath exports (e.g. `@agents/shared/git`). No barrel index.

## Key server source files

```text
apps/server/src/
├── serverLayers.ts                        # Composes all Effect Layers; registers all adapters
├── wsServer.ts                            # WebSocket server; routes JSON-RPC methods
├── claudeCodeAppServerManager.ts          # Claude Code session manager (per-turn subprocess)
├── claudeCodeAppServerSession.ts          # Claude Code session types & environment helpers
├── claudeCodeAppServerHelpers.ts          # Claude Code pure utilities (tool classification, control protocol)
├── provider/
│   ├── Layers/
│   │   ├── ClaudeCodeAdapter.ts           # Claude Code Effect adapter layer
│   │   ├── CodexAdapter.ts                # Codex adapter layer
│   │   ├── GeminiAdapter.ts               # Gemini adapter layer
│   │   ├── ProviderAdapterRegistry.ts     # In-memory adapter registry
│   │   └── ProviderService.ts             # Routes calls to registered adapters
│   ├── Services/
│   │   ├── ClaudeCodeAdapter.ts           # Claude Code service contract
│   │   └── ProviderAdapter.ts             # Generic adapter shape
│   └── claudeCodeCliVersion.ts            # CLI version validation (min v2.0.0)
├── orchestration/                         # Domain event engine & projections
├── checkpointing/                         # Diff tracking per turn
├── git/                                   # Git operations
├── terminal/                              # PTY management (Bun or node-pty)
└── persistence/                           # SQLite-backed event store & repositories
```

## Key web source files

```text
apps/web/src/
├── store.ts                               # Zustand state (projects, threads, messages)
├── session-logic.ts                       # Session lifecycle (connecting → ready → running)
├── types.ts                               # TypeScript interfaces
├── nativeApi.ts                           # WebSocket RPC client (interface)
├── wsNativeApi.ts                         # WebSocket transport implementation
├── wsTransport.ts                         # Low-level WebSocket transport
├── appSettings.ts                         # App settings state
├── desktop-bridge.ts                      # Electron DesktopBridge IPC integration
├── router.ts                              # TanStack Router instance
├── routes/
│   ├── __root.tsx
│   ├── _chat.tsx
│   ├── _chat.index.tsx
│   ├── _chat.$threadId.tsx
│   └── _chat.settings.tsx                 # Provider & model selection UI
└── components/
    ├── ChatView.tsx
    ├── ChatViewComposerArea.tsx
    ├── ChatViewMessageList.tsx
    ├── ChatViewToolbar.tsx
    ├── Sidebar.tsx
    ├── BranchToolbar.tsx
    ├── DiffPanel.tsx
    ├── GitActionsControl.tsx
    ├── PlanSidebar.tsx
    ├── ProjectDock.tsx
    ├── ThreadTerminalDrawer.tsx
    └── settings/                          # Per-provider settings components
```
