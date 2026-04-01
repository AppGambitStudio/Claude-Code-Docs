# Claude Code Documentation

Welcome to the documentation for **Claude Code** (`claude-code-main`). This repository contains the source code for an agentic coding assistant CLI developed by Anthropic, which brings the conversational abilities of Claude natively into your terminal environment.

## Overview

Claude Code is a robust TypeScript project built for execution in the terminal using Node/Bun runtimes and Ink/React for UI rendering. Its primary purpose is to help users manage codebases by executing bash sequences, reading/writing files, orchestrating complex tools, and interfacing natively with the Model Context Protocol (MCP).

The codebase contains **1,884+ TypeScript/TSX files** organized across **101 commands**, **43 tools**, **147 React components**, **85+ hooks**, **329+ utilities**, and **38 service modules**.

## Table of Contents

### Core Architecture

1. **[Architecture Summary](./summary.md)**
   High-level overview of all components, technology stack, and how they interact.

2. **[Core Architecture & Orchestration](./architecture.md)**
   The `QueryEngine` message loop, API services, dynamic system prompts, state management, and FastMode.

3. **[CLI & Terminal UI](./cli_and_ui.md)**
   Command-line entry points, React/Ink terminal UI, interactive helpers, and the slash command system.

### Tool & Agent Ecosystem

4. **[Tools & Agent Ecosystem](./tools_and_agents.md)**
   Native tool primitives (Bash, File I/O, Grep, Glob), advanced tools (LSP, MCP), and the AgentTool architecture.

5. **[Agent Swarms & Teams](./agent_swarms_and_teams.md)**
   Multi-agent orchestration with tmux/iTerm2/in-process backends, team management, coordinator mode, inter-agent messaging, swarm architecture, and team memory sync.

### Services & Integrations

6. **[Services & Integrations](./services.md)**
   MCP backend, session analytics, cost tracking, voice input, and OAuth management.

7. **[Bridge & Remote Execution](./bridge_and_remote.md)**
   Remote session lifecycle, transport layers (v1 WebSocket, v2 SSE), bridge messaging protocol, trusted device management, and the REPL bridge.

### Security & Permissions

8. **[Permission & Security System](./permissions_and_security.md)**
   Permission modes, rule sources and matching, the multi-stage checking flow (rules -> classifier -> interactive), secure storage, auto mode, and permission persistence.

9. **[Authentication, State & Configuration](./auth_state_and_config.md)**
   OAuth 2.0 + PKCE authentication, API key management, AppState architecture, settings hierarchy, remote managed settings, user hooks system, and keybindings.

### Session & Memory

10. **[Session, History & Memory](./session_history_and_memory.md)**
    JSONL session storage, resume/rewind, prompt history, context compaction strategies, the memory directory system, session memory, and background memory extraction.

### Extensibility

11. **[Plugins & Skills System](./plugins_and_skills.md)**
    Plugin types and discovery, marketplace integration, plugin lifecycle, the skills architecture (inline vs fork execution), bundled skills, and extension points.

### Planning, Tasks & IDE

12. **[Planning, Tasks & IDE Integration](./planning_tasks_and_ide.md)**
    Planning mode workflow, git worktree isolation, the task system (CRUD, dependencies, ownership), IDE integration (VS Code, JetBrains, Chrome extension), and context management.

## Reference Links

- [Model Context Protocol (MCP) Specification](https://modelcontextprotocol.io)
- [Anthropic API Documentation](https://docs.anthropic.com)
