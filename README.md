# Claude Code Documentation

Welcome to the documentation for **Claude Code** (`claude-code-main`). This repository contains the source code for an agentic coding assistant CLI developed by Anthropic, which brings the conversational abilities of Claude natively into your terminal environment.

## Overview

Claude Code is a robust TypeScript project built for execution in the terminal using Node/Bun runtimes and Ink/React for UI rendering. Its primary purpose is to help users manage codebases by executing bash sequences, reading/writing files, orchestrating complex tools, and interfacing natively with the Model Context Protocol (MCP).

The codebase contains **1,884+ TypeScript/TSX files** organized across **101 commands**, **43 tools**, **147 React components**, **85+ hooks**, **329+ utilities**, and **38 service modules**.
## How Claude Code Works

```
                                    CLAUDE CODE ARCHITECTURE
  ===========================================================================================

  USER INPUT                         CORE ENGINE                         EXTERNAL SERVICES
  ─────────                          ───────────                         ─────────────────

  ┌─────────────┐              ┌────────────────────────┐
  │  Terminal   │              │      main.tsx          │
  │  (stdin)    │──────────►   │  ┌─────────────────┐   │
  │             │              │  │  Commander.js   │   │
  │  - Text     │              │  │  CLI Parser     │   │
  │  - Voice    │              │  └───────┬─────────┘   │
  │  - Images   │              │          │             │
  │  - Files    │              │          ▼             │
  └─────────────┘              │  ┌─────────────────┐   │
        ▲                      │  │  setup.ts       │   │
        │                      │  │  Bootstrap      │   │
        │                      │  │  (12 steps)     │   │
        │                      │  └───────┬─────────┘   │
        │                      └──────────┼─────────────┘
        │                                 │
        │                                 ▼
        │                      ┌──────────────────────────────────────────────────┐
        │                      │              QueryEngine.ts                      │
        │                      │  ┌─────────────────────────────────────────────┐ │
        │                      │  │            MESSAGE LOOP                     │ │
        │                      │  │                                             │ │
        │                      │  │  1. Build System Prompt                     │ │
        │                      │  │     ├─ CLAUDE.md files                      │ │
        │                      │  │     ├─ Memory (memdir/)                     │ │
        │                      │  │     ├─ Git context                          │ │
        │                      │  │     └─ Environment info                     │ │
        │                      │  │                     │                       │ │
        │                      │  │  2. Call Anthropic API ─────────────────────┼─┼──► Anthropic API
        │                      │  │     (streaming)     │                       │ │    (claude.ts)
        │                      │  │                     ▼                       │ │
        │                      │  │  3. Parse Stream Response                   │ │
        │                      │  │     ├─ Text chunks ─────────► stdout        │ │
        │                      │  │     └─ Tool calls ──┐                       │ │
        │                      │  │                     │                       │ │
        │                      │  │  4. Execute Tools   │                       │ │
        │                      │  │         ┌───────────┘                       │ │
        │                      │  │         ▼                                   │ │
        │                      │  │  ┌─────────────┐    ┌─────────────────────┐ │ │
        │                      │  │  │ Permission  │    │   TOOL POOL (43)    │ │ │
        │                      │  │  │ Check       │───►│                     │ │ │
        │                      │  │  │             │    │  BashTool           │ │ │
        │                      │  │  │ ┌─────────┐ │    │  FileRead/Edit/Write│ │ │
        │                      │  │  │ │ Rules   │ │    │  Grep / Glob        │ │ │
        │                      │  │  │ │ Classifr│ │    │  WebSearch/Fetch    │ │ │
        │                      │  │  │ │ Interact│ │    │  MCPTool / LSPTool  │ │ │
        │                      │  │  │ └─────────┘ │    │  AgentTool          │ │ │
        │                      │  │  └─────────────┘    │  TaskCreate/Update  │ │ │
        │                      │  │         │           │  EnterPlanMode      │ │ │
        │                      │  │         │           │  EnterWorktree      │ │ │
        │                      │  │         │           │  SkillTool          │ │ │
        │                      │  │         │           │  NotebookEdit       │ │ │
        │                      │  │         │           └─────────────────────┘ │ │
        │                      │  │         │                                   │ │
        │                      │  │  5. Append Tool Results to Messages         │ │
        │                      │  │         │                                   │ │
        │                      │  │         └──────► Loop back to step 2        │ │
        │                      │  │                  (until no more tool calls) │ │
        │                      │  └─────────────────────────────────────────────┘ │
        │                      │                                                  │
        │                      │  Cost Tracker ─── Token counting + pricing       │
        │                      │  Session Store ── JSONL transcript persistence   │
        │                      └──────────────────────────────────────────────────┘
        │                                 │
        │                                 ▼
        │                      ┌──────────────────────┐
        │                      │    Ink / React UI    │
        │                      │                      │
        │                      │  ┌─────────────────┐ │
        │                      │  │ App.tsx         │  │
        └──────────────────────┤  │  ├─ StatusLine  │  │
                               │  │  ├─ Messages    │  │
                               │  │  ├─ ToolApproval│  │
                               │  │  ├─ Dialogs     │  │
                               │  │  └─ TextInput   │  │
                               │  └─────────────────┘  │
                               └──────────────────────┘


  ===========================================================================================
                              EXTENSION & INTEGRATION SYSTEMS
  ===========================================================================================

  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐   ┌──────────────────────────┐
  │   PLUGINS    │   │   SKILLS     │   │    MCP       │   │     BRIDGE / REMOTE      │
  │              │   │              │   │  SERVERS     │   │                          │
  │ Marketplace  │   │ Bundled      │   │              │   │  ┌────────┐  ┌────────┐  │
  │ User plugins │   │ User skills  │   │ Stdio/SSE/WS │   │  │claude  │  │ Local  │  │
  │ Built-in     │   │ Plugin skills│   │ transports   │   │  │ .ai    │──│ Bridge │  │
  │              │   │              │   │              │   │  └────────┘  └───┬────┘  │
  │ Hooks ───────┤   │ Inline/Fork  │   │ Tool/Resource│   │                  │       │
  │ Commands     │   │ execution    │   │ discovery    │   │  Poll / SSE / WebSocket  │
  │ MCP servers  │   │              │   │              │   │                  │       │
  │ LSP servers  │   │              │   │              │   │  ┌───────────────┘       │
  └──────────────┘   └──────────────┘   └──────────────┘   │  ▼                       │
                                                           │  Session Runner          │
                                                           │  (child CLI process)     │
                                                           └──────────────────────────┘

  ┌────────────────────────────────────────────────────────────────────────────────────┐
  │                          AGENT SWARMS & TEAMS                                      │
  │                                                                                    │
  │   ┌───────────┐    SendMessage     ┌───────────┐          ┌───────────┐            │
  │   │  Leader   │◄──────────────────►│ Worker 1  │          │ Worker 2  │            │
  │   │ (coord.)  │    Mailbox         │ (tmux)    │          │ (in-proc) │            │
  │   │           │                    │           │          │           │            │
  │   │ TeamFile  │    ┌───────────┐   │ Worktree  │          │ AsyncLocal│            │
  │   │ AppState  │    │ Worker 3  │   │ isolated  │          │ Storage   │            │
  │   └───────────┘    │ (iTerm2)  │   └───────────┘          └───────────┘            │
  │                    └───────────┘                                                   │
  │   Backends: tmux | iTerm2 | in-process                                             │
  └────────────────────────────────────────────────────────────────────────────────────┘


  ===========================================================================================
                              PERSISTENCE & SECURITY
  ===========================================================================================

  ┌──────────────────────┐   ┌───────────────────────┐   ┌──────────────────────────┐
  │   SESSION STORAGE    │   │   CONFIGURATION       │   │      SECURITY            │
  │                      │   │                       │   │                          │
  │ ~/.claude/projects/  │   │ ~/.claude/            │   │  OAuth 2.0 + PKCE        │
  │   <project>/         │   │   settings.json       │   │  Keychain (macOS)        │
  │   <session>.jsonl    │   │   keybindings.json    │   │  Permission rules        │
  │                      │   │                       │   │  Auto-mode classifier    │
  │ ~/.claude/           │   │ .claude/              │   │  Trusted devices         │
  │   history.jsonl      │   │   settings.json       │   │  Secret scanning         │
  │                      │   │   CLAUDE.md           │   │  Sandbox (seatbelt)      │
  │ Memory:              │   │   settings.local.json │   │                          │
  │   memory/MEMORY.md   │   │                       │   │  7 permission modes:     │
  │   memory/*.md        │   │ Remote managed        │   │  default, acceptEdits,   │
  │                      │   │   settings (API)      │   │  plan, bypass, dontAsk,  │
  │ Plans:               │   │                       │   │  auto, bubble            │
  │   plans/*.md         │   │ MDM / policy drops    │   │                          │
  └──────────────────────┘   └───────────────────────┘   └──────────────────────────┘


  ===========================================================================================
                              IDE INTEGRATION
  ===========================================================================================

  ┌────────────┐  MCP   ┌──────────────┐  Native    ┌──────────────┐
  │  VS Code   │◄──────►│              │  Messaging │   Chrome     │
  │  JetBrains │  SSE/  │  Claude Code │◄──────────►│  Extension   │
  │            │  WS    │   CLI        │            │              │
  └────────────┘        │              │  Handoff   ┌──────────────┐
                        │              │◄──────────►│ Claude       │
                        └──────────────┘            │ Desktop App  │
                                                    └──────────────┘
```

## Table of Contents

### Core Architecture

1. **[Core Architecture & Orchestration](./architecture.md)**
   The `QueryEngine` message loop, API services, dynamic system prompts, cost tracking, state management, and FastMode.

2. **[CLI & Terminal UI](./cli_and_ui.md)**
   Command-line entry points, React/Ink terminal UI, interactive helpers, dialog system, and all 101+ slash commands.

### Tool & Agent Ecosystem

3. **[Tools & Agent Ecosystem](./tools_and_agents.md)**
   All 43 tools, the `ToolDef` interface, file system tools, web tools, protocol tools (MCP, LSP), and permission validation.

4. **[Agent Swarms & Teams](./agent_swarms_and_teams.md)**
   Multi-agent orchestration with tmux/iTerm2/in-process backends, team management, coordinator mode, inter-agent messaging, swarm architecture, and team memory sync.

### Services & Integrations

5. **[Services & Integrations](./services.md)**
   MCP backend, session analytics, cost tracking, voice I/O, rate limiting, compaction, LSP, and prompt suggestions.

6. **[Bridge & Remote Execution](./bridge_and_remote.md)**
   Remote session lifecycle, transport layers (v1 WebSocket, v2 SSE), bridge messaging protocol, trusted device management, and the REPL bridge.

### Security & Permissions

7. **[Permission & Security System](./permissions_and_security.md)**
   Permission modes, rule sources and matching, the multi-stage checking flow (rules -> classifier -> interactive), secure storage, auto mode, and permission persistence.

8. **[Authentication, State & Configuration](./auth_state_and_config.md)**
   OAuth 2.0 + PKCE authentication, API key management, AppState architecture, settings hierarchy, remote managed settings, user hooks system, and keybindings.

### Session & Memory

9. **[Session, History & Memory](./session_history_and_memory.md)**
   JSONL session storage, resume/rewind, prompt history, context compaction strategies, the memory directory system, session memory, and background memory extraction.

### Extensibility

10. **[Plugins & Skills System](./plugins_and_skills.md)**
    Plugin types and discovery, marketplace integration, plugin lifecycle, the skills architecture (inline vs fork execution), bundled skills, and extension points.

### Planning, Tasks & IDE

11. **[Planning, Tasks & IDE Integration](./planning_tasks_and_ide.md)**
    Planning mode workflow, git worktree isolation, the task system (CRUD, dependencies, ownership), IDE integration (VS Code, JetBrains, Chrome extension), and context management.

## Reference Links

- [Model Context Protocol (MCP) Specification](https://modelcontextprotocol.io)
- [Anthropic API Documentation](https://docs.anthropic.com)
