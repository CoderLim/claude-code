# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

This is a research mirror of Claude Code's leaked source code (discovered March 2026 via npm sourcemap). It is **not a runnable project** — there is no `package.json`, no build toolchain, and no entry point configured. The source is here for study and reference only.

## Architecture

The codebase is a TypeScript/React CLI application built on [Ink](https://github.com/vadimdemedes/ink) (React for terminal UIs) and Bun.

### Entry Point

`src/main.tsx` — CLI entry point using Commander.js. Handles startup profiling, MDM config reads, keychain prefetch, auth, feature flags (GrowthBook), and launches the REPL via `src/replLauncher.tsx`.

### Core Query Loop

`src/QueryEngine.ts` — The central LLM orchestration engine. Receives messages, resolves tools, calls `src/query.ts` (raw Anthropic SDK calls), handles tool use cycles, permission checks, memory loading, and usage tracking.

`src/query.ts` — Thin wrapper around the Anthropic SDK that executes a single streaming API call.

### Tools (`src/tools/`)

40+ tools, each in its own subdirectory. Key ones:
- `BashTool` — shell execution
- `FileReadTool`, `FileEditTool`, `FileWriteTool`, `GlobTool`, `GrepTool` — file operations
- `AgentTool` — spawns subagents (multi-agent orchestration)
- `MCPTool` — MCP server integration
- `SkillTool` — loads and executes skills
- `EnterPlanModeTool` / `ExitPlanModeTool` — plan mode
- `EnterWorktreeTool` / `ExitWorktreeTool` — git worktree isolation
- `TaskCreateTool`, `SendMessageTool` — team/task coordination

Base type definitions are in `src/Tool.ts`.

### Services (`src/services/`)

- `api/` — Anthropic SDK wrappers, usage logging, rate limits, error handling
- `mcp/` — MCP server lifecycle and connection management
- `autoDream/` — Background memory consolidation subagent (reads `MEMORY.md`, gathers signals from daily logs, updates durable memory, prunes context)
- `extractMemories/` — Memory extraction pipeline
- `oauth/` — Authentication
- `analytics/` — GrowthBook feature flags, telemetry
- `compact/` — Context compaction logic
- `lsp/` — Language server protocol integration
- `policyLimits/` — Usage policy enforcement

### Multi-Agent (`src/coordinator/`, `src/remote/`)

`src/coordinator/coordinatorMode.ts` — "Swarm" orchestration for parallel subagents.

`src/remote/` — WebSocket-based remote session management (`RemoteSessionManager.ts`, `SessionsWebSocket.ts`), used by the bridge/IDE integrations.

### Bridge (`src/bridge/`)

IDE integration layer (VS Code, JetBrains). Communicates over a local bridge API. Key files: `bridgeMain.ts`, `bridgeApi.ts`, `replBridge.ts`.

### Buddy (`src/buddy/`)

Tamagotchi-style terminal companion. Uses Mulberry32 PRNG seeded from `userId` for deterministic gacha (18 species, Common → Legendary). Hidden feature, not surfaced in normal CLI usage.

### Notable Subsystems

- **KAIROS** — always-on proactive assistant that watches logs and acts autonomously
- **ULTRAPLAN** — offloads complex tasks to a remote Opus 4.6 session for up to 30 minutes of deep planning
- **Undercover Mode** (`src/utils/undercover.ts`) — prevents leaking internal codenames when Anthropic employees use the tool on public repos
- **Memory system** (`src/memdir/`) — file-based persistent memory at `~/.claude/`
- **Startup profiler** (`src/utils/startupProfiler.ts`) — perf checkpoints from `main.tsx` entry through REPL launch

## Key Patterns

- Tools are defined with a `Tool` base type (input schema via Zod, `call()` method, permission metadata).
- `QueryEngine` drives the agentic loop: send messages → receive tool calls → execute tools → append results → repeat until no more tool calls.
- Permission modes (`auto`, `default`, `bypassPermissions`, etc.) gate which tools can run without user confirmation.
- Context is managed via `src/context.ts`; app state via `src/state/AppState.js`.
