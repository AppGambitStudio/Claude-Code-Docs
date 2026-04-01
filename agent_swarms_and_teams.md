# Agent Swarms & Teams

This document describes the multi-agent architecture in Claude Code, covering how agents are spawned, how they communicate, how teams are formed and managed, and the supporting infrastructure for coordination, memory, and summarization.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Agent Tool](#2-agent-tool)
3. [Inter-Agent Communication](#3-inter-agent-communication)
4. [Team Management](#4-team-management)
5. [Coordinator Mode](#5-coordinator-mode)
6. [Swarm Architecture](#6-swarm-architecture)
7. [Agent Summaries](#7-agent-summaries)
8. [Team Memory Sync](#8-team-memory-sync)
9. [Feature Flags & Configuration](#9-feature-flags--configuration)
10. [Key Source Files Reference](#10-key-source-files-reference)

---

## 1. Overview

Claude Code supports a multi-agent architecture where a primary agent (the "leader" or "coordinator") can spawn, manage, and communicate with sub-agents to perform complex tasks in parallel. The system operates in two distinct modes:

- **Standard mode**: The main agent spawns sub-agents via the `Agent` tool. Sub-agents run autonomously, complete a task, and return a result. Communication is one-directional (parent to child) unless the swarm features are enabled.
- **Coordinator mode**: Activated via `CLAUDE_CODE_COORDINATOR_MODE=1`. The main agent becomes a coordinator that exclusively orchestrates workers. Workers are spawned as background agents and report back via `<task-notification>` messages. The coordinator synthesizes results and communicates with the user.

When agent swarms/teams are enabled, agents can form persistent teams with named members, use `SendMessage` for bidirectional messaging, share task lists, and coordinate through mailbox-based communication.

### Key Concepts

| Concept | Description |
|---------|-------------|
| **Sub-agent** | A child agent spawned via the `Agent` tool, runs autonomously and returns a result |
| **Fork** | A sub-agent that inherits the parent's full conversation context (experiment-gated) |
| **Teammate** | A named agent in a team, addressable via `SendMessage`, persists across turns |
| **Team lead** | The agent that created the team, receives idle notifications from teammates |
| **Coordinator** | A specialized lead that only orchestrates workers (no direct tool use) |
| **Worker** | An agent spawned by a coordinator in coordinator mode |

---

## 2. Agent Tool

The `Agent` tool (`tools/AgentTool/AgentTool.tsx`) is the primary mechanism for spawning sub-agents. It is registered under the name `Agent` (legacy name: `Task`).

### Input Schema

The tool accepts these parameters:

| Parameter | Type | Description |
|-----------|------|-------------|
| `prompt` | string (required) | The task for the agent to perform |
| `description` | string (required) | Short 3-5 word description of the task |
| `subagent_type` | string (optional) | Specialized agent type to use (e.g., `general-purpose`, `Explore`, `Plan`) |
| `model` | `sonnet` / `opus` / `haiku` (optional) | Model override for this agent |
| `run_in_background` | boolean (optional) | Run agent asynchronously; caller is notified on completion |
| `name` | string (optional) | Name for the agent, making it addressable via `SendMessage` (swarms only) |
| `team_name` | string (optional) | Team to spawn the agent into (swarms only) |
| `mode` | PermissionMode (optional) | Permission mode for the teammate (swarms only) |
| `isolation` | `worktree` / `remote` (optional) | Isolation mode: git worktree or remote environment |
| `cwd` | string (optional) | Working directory override (Kairos feature) |

### Built-In Agent Types

Built-in agents are defined in `tools/AgentTool/builtInAgents.ts` and the `tools/AgentTool/built-in/` directory:

| Agent Type | Purpose | Model | Key Restrictions |
|------------|---------|-------|------------------|
| `general-purpose` | Multi-step tasks, code search, research | Default subagent model | Full tool access (`tools: ['*']`) |
| `Explore` | Fast read-only codebase exploration | `haiku` (external), `inherit` (internal) | No file editing tools; disallows Agent, Edit, Write, NotebookEdit |
| `Plan` | Planning and analysis | Varies | Read-only, similar restrictions to Explore |
| `verification` | Tries to break implementations | Varies | Cannot modify project files; can write to `/tmp` |
| `claude-code-guide` | Help with Claude Code usage | Default | Non-SDK entrypoints only |
| `statusline-setup` | UI configuration | Default | Utility agent |

The `Explore` and `Plan` agents are one-shot built-in types (defined in `ONE_SHOT_BUILTIN_AGENT_TYPES`), which skip the agentId/SendMessage/usage trailer to save tokens.

### Custom Agents

Users can define custom agents via markdown files in `.claude/agents/` (project), `~/.claude/agents/` (user), or via managed/plugin settings. The frontmatter schema (`loadAgentsDir.ts`) supports:

- `description`, `prompt` (required)
- `tools` / `disallowedTools` - tool allowlist/denylist
- `model` - model override or `inherit`
- `effort` - effort level
- `permissionMode` - permission handling mode
- `mcpServers` - agent-specific MCP server connections
- `hooks` - lifecycle hooks
- `maxTurns` - maximum conversation turns
- `memory` - persistent memory scope (`user`, `project`, `local`)
- `isolation` - `worktree` or `remote`
- `background` - default to background execution

Agent sources are prioritized: user > project > local > managed > plugin > CLI arg > built-in. Higher-priority sources override lower-priority agents with the same type name.

### Agent Execution Flow

The core execution happens in `runAgent.ts`:

1. Agent-specific MCP servers are initialized (if defined in frontmatter)
2. A subagent context is created via `createSubagentContext()`, isolating file state caches and content replacement state
3. The agent's system prompt is built, incorporating the agent definition's custom prompt, environment details, and optional persistent memory
4. Tool pool is assembled and filtered: `ALL_AGENT_DISALLOWED_TOOLS` are removed, custom agents additionally lose `CUSTOM_AGENT_DISALLOWED_TOOLS`, and async agents are restricted to `ASYNC_AGENT_ALLOWED_TOOLS`
5. The `query()` function drives the agent's conversation loop
6. On completion, agent transcripts are recorded and MCP servers cleaned up

### Fork Sub-Agents

When the `FORK_SUBAGENT` feature flag is active (and not in coordinator mode or non-interactive sessions), omitting `subagent_type` creates a **fork** instead of a fresh agent (`forkSubagent.ts`).

Key fork properties:
- Inherits the parent's full conversation context and system prompt (byte-exact for cache sharing)
- All fork children produce byte-identical API request prefixes for prompt cache sharing
- Tool results are replaced with a uniform placeholder (`"Fork started -- processing in background"`)
- A per-child directive is appended as the final text block
- Fork children cannot recursively fork (detected via `FORK_BOILERPLATE_TAG` in conversation history)
- The `FORK_AGENT` definition uses `tools: ['*']`, `permissionMode: 'bubble'`, `model: 'inherit'`

Fork child messages include strict instructions: no sub-agent spawning, no conversing, no editorializing. Output must follow a structured format (Scope, Result, Key files, Files changed, Issues).

### Agent Memory

Agents with `memory` configured in their definition get persistent memory across sessions (`agentMemory.ts`):

| Scope | Location | Purpose |
|-------|----------|---------|
| `user` | `~/.claude/agent-memory/<agentType>/` | Cross-project learnings |
| `project` | `<cwd>/.claude/agent-memory/<agentType>/` | Project-specific, shared via VCS |
| `local` | `<cwd>/.claude/agent-memory-local/<agentType>/` | Project-specific, not in VCS |

Memory is loaded via `buildMemoryPrompt()` at agent spawn time. The entrypoint file is `MEMORY.md` within the memory directory.

Agent memory snapshots (`agentMemorySnapshot.ts`) support initializing local memory from a committed snapshot in `.claude/agent-memory-snapshots/<agentType>/snapshot.json`. This enables teams to bootstrap agent memory from version-controlled seeds.

### Agent Colors

Each non-general-purpose agent type is assigned a unique color from a palette of 8 colors (red, blue, green, yellow, purple, orange, pink, cyan) defined in `agentColorManager.ts`. Colors are stored in a global map and mapped to theme-specific color keys for UI rendering.

---

## 3. Inter-Agent Communication

### SendMessage Tool

The `SendMessage` tool (`tools/SendMessageTool/SendMessageTool.ts`) enables agents to send messages to each other. It is enabled only when agent swarms are active (`isAgentSwarmsEnabled()`).

**Input schema:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `to` | string (required) | Recipient: teammate name, `"*"` for broadcast, or UDS/bridge address |
| `message` | string or structured object | Message content |
| `summary` | string (optional) | 5-10 word preview shown in UI |

**Message routing:**

Messages are delivered via a file-based mailbox system (`utils/teammateMailbox.ts`). The flow:

1. Sender calls `SendMessage` with a recipient name
2. The tool writes the message to the recipient's mailbox file on disk
3. Recipients poll their mailbox and receive messages as conversation turns
4. For broadcasts (`to: "*"`), the message is written to every teammate's mailbox

**Structured message types:**

The tool supports three structured message types via discriminated union:

- `shutdown_request` - Request a teammate to shut down
- `shutdown_response` - Approve/reject a shutdown request (includes `request_id` and `approve` boolean)
- `plan_approval_response` - Approve/reject a plan (includes `request_id`, `approve`, and optional `feedback`)

**Cross-session messaging** (UDS_INBOX feature): When enabled, messages can be sent to other Claude sessions via Unix domain sockets (`uds:/path/to.sock`) or Remote Control peers (`bridge:session_id`).

**Coordinator mode**: In coordinator mode, `SendMessage` is used to continue existing workers by sending follow-up messages to their agent ID.

---

## 4. Team Management

### TeamCreate Tool

The `TeamCreate` tool (`tools/TeamCreateTool/TeamCreateTool.ts`) creates a new team for multi-agent coordination. It is enabled only when `isAgentSwarmsEnabled()` returns true.

**Input:**
- `team_name` (required) - Name for the team
- `description` (optional) - Team purpose
- `agent_type` (optional) - Role of the team lead

**What it creates:**

1. A team configuration file at `~/.claude/teams/{team-name}/config.json`
2. A corresponding task list directory at `~/.claude/tasks/{team-name}/`
3. Team context in AppState with lead agent information

**Team file structure** (`TeamFile` type in `teamHelpers.ts`):

```typescript
{
  name: string
  description?: string
  createdAt: number
  leadAgentId: string
  leadSessionId?: string
  teamAllowedPaths?: TeamAllowedPath[]
  members: Array<{
    agentId: string
    name: string
    agentType?: string
    model?: string
    color?: string
    planModeRequired?: boolean
    joinedAt: number
    tmuxPaneId: string
    cwd: string
    worktreePath?: string
    sessionId?: string
    subscriptions: string[]
    backendType?: BackendType  // 'tmux' | 'iterm2' | 'in-process'
    isActive?: boolean
    mode?: PermissionMode
  }>
}
```

A leader can only manage one team at a time. If a team name already exists, a unique name is auto-generated via `generateWordSlug()`.

### TeamDelete Tool

The `TeamDelete` tool (`tools/TeamDeleteTool/TeamDeleteTool.ts`) cleans up team resources:

1. Checks for active (non-idle) members and refuses deletion if any are still running
2. Removes the team directory and task directory
3. Clears teammate color assignments
4. Clears team context from AppState

The tool takes no input parameters -- it operates on the current session's team context.

### Team Workflow

The intended lifecycle as described in the TeamCreate prompt:

1. **Create team** with `TeamCreate`
2. **Create tasks** via Task tools (TaskCreate, TaskList, etc.) -- they auto-use the team's task list
3. **Spawn teammates** via `Agent` tool with `team_name` and `name` parameters
4. **Assign tasks** using `TaskUpdate` with `owner` to give tasks to idle teammates
5. **Teammates work** and mark tasks completed via `TaskUpdate`
6. **Idle cycle** -- teammates go idle after each turn and send idle notifications automatically
7. **Shutdown** -- leader sends `shutdown_request` via `SendMessage` to each teammate
8. **Cleanup** -- `TeamDelete` removes team and task directories

### Teammate Identity

Teammate identity is managed through `utils/teammate.ts` with a priority system:

1. **AsyncLocalStorage** (in-process teammates) -- via `teammateContext.ts`
2. **dynamicTeamContext** (tmux/iTerm2 teammates via CLI args)

Key identity functions:
- `getAgentId()` / `getAgentName()` / `getTeamName()` -- resolve current agent identity
- `isTeammate()` -- returns true if running as a teammate (requires both agentId and teamName)
- `isTeamLead()` -- checks if this session created the team
- `isPlanModeRequired()` -- checks if plan approval is needed before implementation
- `getTeammateColor()` -- returns assigned color for UI rendering

---

## 5. Coordinator Mode

Coordinator mode (`coordinator/coordinatorMode.ts`) is a specialized orchestration mode activated by `CLAUDE_CODE_COORDINATOR_MODE=1`. It transforms the main agent into a pure coordinator that delegates all tool-based work to workers.

### Activation

```typescript
function isCoordinatorMode(): boolean {
  if (feature('COORDINATOR_MODE')) {
    return isEnvTruthy(process.env.CLAUDE_CODE_COORDINATOR_MODE)
  }
  return false
}
```

When resuming sessions, `matchSessionMode()` reconciles the current mode with the session's stored mode, flipping the environment variable if needed.

### Coordinator System Prompt

The coordinator receives a specialized system prompt (`getCoordinatorSystemPrompt()`) that defines:

- **Role**: Orchestrator that directs workers, synthesizes results, and communicates with the user
- **Available tools**: `Agent` (spawn workers), `SendMessage` (continue workers), `TaskStop` (stop workers), plus optional PR activity subscription
- **Worker result format**: Workers report via `<task-notification>` XML containing task-id, status, summary, result, and usage metrics

### Worker Context

`getCoordinatorUserContext()` provides workers with information about their available tools. This includes:
- The list of tools from `ASYNC_AGENT_ALLOWED_TOOLS` (minus internal tools like TeamCreate, TeamDelete, SendMessage, SyntheticOutput)
- MCP server tool availability
- Scratchpad directory for cross-worker knowledge sharing (when the `tengu_scratch` gate is enabled)

### Task Phases

The coordinator prompt defines a structured workflow:

| Phase | Who | Purpose |
|-------|-----|---------|
| Research | Workers (parallel) | Investigate codebase, find files, understand problem |
| Synthesis | Coordinator | Read findings, craft implementation specs |
| Implementation | Workers | Make targeted changes per spec, commit |
| Verification | Workers | Test changes work |

Key principles:
- Parallelism is emphasized: launch independent workers concurrently
- Read-only tasks run in parallel freely; write-heavy tasks are serialized per file set
- The coordinator must synthesize research findings before directing implementation (never write "based on your findings")
- Workers can be continued via `SendMessage` (preserves context) or replaced with fresh agents

---

## 6. Swarm Architecture

### Backends

The swarm system supports three backend types for running teammates, defined in `utils/swarm/backends/types.ts`:

| Backend | Type | Description |
|---------|------|-------------|
| **TmuxBackend** | `tmux` | Uses tmux for pane management. Works inside existing tmux sessions or creates external sessions |
| **ITermBackend** | `iterm2` | Uses iTerm2 native split panes via the `it2` CLI |
| **InProcessBackend** | `in-process` | Runs in the same Node.js process with AsyncLocalStorage isolation |

**Backend detection priority** (`registry.ts`):

1. If inside tmux, always use tmux (even in iTerm2)
2. If in iTerm2 with `it2` CLI available, use iTerm2
3. If in iTerm2 without `it2`, indicate setup is needed
4. If tmux is available, use tmux (creates external session)
5. Fall back to in-process backend

The detection result is cached for the process lifetime. Backends implement the `PaneBackend` interface for pane-based backends or the `TeammateExecutor` interface.

### In-Process Teammates

In-process teammates (`utils/swarm/inProcessRunner.ts`, `utils/swarm/spawnInProcess.ts`) run within the same Node.js process:

- **Context isolation**: `AsyncLocalStorage` provides per-teammate identity via `runWithTeammateContext()`
- **Resource sharing**: Share API client, MCP connections with the leader
- **Communication**: File-based mailbox (same as pane-based teammates)
- **Termination**: Via `AbortController` (independent from parent)
- **Permission handling**: Uses the leader's `ToolUseConfirm` dialog via `leaderPermissionBridge.ts`, with worker badge. Falls back to mailbox-based permission requests when the bridge is unavailable.

The spawn flow:
1. `spawnInProcessTeammate()` creates context, registers task in AppState
2. `startInProcessTeammate()` starts the agent execution loop in the background
3. The runner wraps `runAgent()` with teammate context, progress tracking, idle notifications, plan mode support, and auto-compaction

### Pane-Based Teammates (Tmux/iTerm2)

For tmux and iTerm2 backends:
- Teammates are spawned as separate Claude Code processes in terminal panes
- The teammate command is determined by `getTeammateCommand()` (defaults to current executable, overridable via `CLAUDE_CODE_TEAMMATE_COMMAND`)
- CLI flags are propagated via `buildInheritedCliFlags()` including permission mode, model selection, plugin configuration, and teammate mode
- Environment variables are forwarded explicitly since tmux may start a new login shell

### Teammate Layout

`teammateLayoutManager.ts` handles visual layout:
- Colors are assigned round-robin from the 8-color palette
- Pane creation delegates to the detected backend
- Leader stays on the left (30%), teammates on the right (70%) in tmux

### Teammate Initialization

When a teammate starts (`teammateInit.ts`):
1. Team-wide allowed paths from the team file are applied as permission rules
2. A `Stop` hook is registered that sends an idle notification to the team leader when the teammate's turn ends
3. The idle notification includes a summary of the last peer DM (for leader visibility)

### Reconnection

`reconnection.ts` handles teammate context restoration:
- **Fresh spawns**: Initialize from CLI args stored in `dynamicTeamContext`
- **Resumed sessions**: Initialize from teamName/agentName stored in the transcript via `initializeTeammateContextFromSession()`

### Teammate System Prompt

All teammates receive an addendum (`teammatePromptAddendum.ts`) appended to their system prompt:

> You are running as an agent in a team. To communicate with anyone on your team, use the SendMessage tool. Just writing a response in text is not visible to others on your team.

### Permission Sync

`permissionSync.ts` provides synchronized permission prompts across swarm agents:

1. Worker encounters a permission prompt
2. Worker sends a `permission_request` to the leader's mailbox (includes tool name, description, input, suggested rules)
3. Leader polls mailbox and detects permission requests
4. User approves/denies via leader's UI
5. Leader sends `permission_response` to worker's mailbox
6. Worker polls and continues execution

For in-process teammates, the `leaderPermissionBridge.ts` provides a more direct path: teammates use the leader's `ToolUseConfirm` dialog with a worker badge, bypassing the mailbox for permission requests.

---

## 7. Agent Summaries

The `AgentSummary` service (`services/AgentSummary/agentSummary.ts`) provides periodic progress summaries for coordinator mode sub-agents.

### How It Works

1. A timer fires every 30 seconds (`SUMMARY_INTERVAL_MS`)
2. The agent's current transcript is read from disk
3. A forked agent call (`runForkedAgent()`) is made with the same `CacheSafeParams` as the parent agent to share the prompt cache
4. The fork receives a prompt asking for a 3-5 word present-tense description of the most recent action
5. Tools are included in the request for cache key matching but denied via `canUseTool` callback
6. The resulting summary is stored on `AgentProgress` for UI display via `updateAgentSummary()`

### Summary Prompt

The summary prompt enforces specific formatting:
- Present tense with -ing verbs
- Name the file or function, not the branch
- 3-5 words maximum
- Must differ from the previous summary

Good examples: "Reading runAgent.ts", "Fixing null check in validate.ts", "Running auth module tests"

### Lifecycle

- `startAgentSummarization()` returns a `{ stop }` handle
- The timer schedules the next summary after each completion (not initiation) to prevent overlapping summaries
- On stop, any in-flight summary is aborted via `AbortController`

---

## 8. Team Memory Sync

The Team Memory Sync service (`services/teamMemorySync/`) synchronizes team memory files between the local filesystem and a server API. Team memory is scoped per-repository (identified by git remote) and shared across all authenticated organization members.

### API Contract

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/claude_code/team_memory?repo={owner/repo}` | GET | Fetch team memory data (includes entryChecksums) |
| `/api/claude_code/team_memory?repo={owner/repo}&view=hashes` | GET | Metadata + checksums only (no entry bodies) |
| `/api/claude_code/team_memory?repo={owner/repo}` | PUT | Upload entries (upsert semantics) |

### Sync Semantics

- **Pull**: Server content overwrites local files (server wins per-key)
- **Push**: Only keys whose content hash differs from `serverChecksums` are uploaded (delta upload). Server uses upsert: keys not in the PUT are preserved
- **Deletions do not propagate**: Deleting a local file will not remove it from the server; the next pull restores it locally

### State Management

All mutable sync state is held in a `SyncState` object:

```typescript
type SyncState = {
  lastKnownChecksum: string | null          // ETag for conditional requests
  serverChecksums: Map<string, string>       // Per-key SHA-256 hashes
  serverMaxEntries: number | null            // Server-enforced cap (learned from 413)
}
```

### File Watcher

`watcher.ts` watches the team memory directory for changes:
- Performs an initial pull on startup
- Uses `fs.watch({ recursive: true })` for file change detection
- Pushes are debounced (2-second delay after last change)
- Permanent failures (4xx except 409/429, no OAuth, no repo) suppress further retries until session restart
- File deletions (`unlink`) clear the suppression flag (recovery action)

### Secret Scanning

Before any upload, `secretScanner.ts` scans content for credentials using a curated subset of high-confidence gitleaks rules. Detected patterns include:
- AWS access tokens, GCP API keys, Azure AD client secrets
- Anthropic API keys, OpenAI API keys
- GitHub PATs and fine-grained tokens
- Stripe, Twilio, SendGrid, Shopify keys
- Private keys (RSA, DSA, EC, PGP)

Files containing detected secrets are skipped during push and reported as `SkippedSecretFile` entries.

The `teamMemSecretGuard.ts` module provides a `checkTeamMemSecrets()` function called from `FileWriteTool` and `FileEditTool` validation to prevent the model from writing secrets into team memory files that would be synced to collaborators.

### Data Types

Team memory content is stored as flat key-value pairs where keys are relative file paths (e.g., `MEMORY.md`, `patterns.md`) and values are UTF-8 string content (typically Markdown). Per-entry size is capped at 250KB. PUT body size is capped at 200KB with larger payloads split into sequential uploads.

---

## 9. Feature Flags & Configuration

### Agent Swarms Gate

The centralized gate for all swarm features is `isAgentSwarmsEnabled()` in `utils/agentSwarmsEnabled.ts`:

| User Type | Requirements |
|-----------|-------------|
| Internal (`ant`) | Always enabled |
| External | Requires opt-in via `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` env var OR `--agent-teams` CLI flag, AND the `tengu_amber_flint` GrowthBook killswitch must be enabled |

### Environment Variables

| Variable | Purpose |
|----------|---------|
| `CLAUDE_CODE_COORDINATOR_MODE=1` | Activates coordinator mode |
| `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` | Enables swarm features for external users |
| `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS` | Disables background agent execution |
| `CLAUDE_AUTO_BACKGROUND_TASKS` | Auto-background agent tasks after 120s |
| `CLAUDE_CODE_TEAMMATE_COMMAND` | Override command for spawning teammate processes |
| `CLAUDE_CODE_AGENT_COLOR` | Set on spawned teammates to indicate assigned color |
| `CLAUDE_CODE_PLAN_MODE_REQUIRED` | Require plan approval before implementation |
| `CLAUDE_CODE_AGENT_LIST_IN_MESSAGES` | Force agent list into attachment messages (cache optimization) |
| `CLAUDE_CODE_REMOTE_MEMORY_DIR` | Override directory for local agent memory persistence |
| `CLAUDE_COWORK_MEMORY_EXTRA_GUIDELINES` | Extra guidelines appended to agent memory prompts |
| `CLAUDE_AGENT_SDK_DISABLE_BUILTIN_AGENTS` | Disable all built-in agents (SDK/API usage) |
| `CLAUDE_CODE_SIMPLE` | Simplifies coordinator worker tools to Bash, Read, Edit only |

### Feature Flags (Build-Time)

| Flag | Purpose |
|------|---------|
| `COORDINATOR_MODE` | Enables coordinator mode code paths |
| `FORK_SUBAGENT` | Enables fork sub-agent feature (context-inheriting forks) |
| `BUILTIN_EXPLORE_PLAN_AGENTS` | Enables Explore and Plan built-in agents |
| `VERIFICATION_AGENT` | Enables the verification agent |
| `KAIROS` | Enables `cwd` parameter on Agent tool |
| `UDS_INBOX` | Enables cross-session messaging via Unix domain sockets |
| `TEAMMEM` | Enables team memory features and secret scanning |
| `BASH_CLASSIFIER` | Enables bash command auto-approval classifier |

### GrowthBook Feature Gates

| Gate | Purpose |
|------|---------|
| `tengu_amber_flint` | Killswitch for external agent swarms |
| `tengu_amber_stoat` | Controls Explore/Plan agent availability |
| `tengu_auto_background_agents` | Auto-background agents after timeout |
| `tengu_agent_list_attach` | Move agent list to attachment messages |
| `tengu_scratch` | Enable scratchpad directory for cross-worker knowledge |
| `tengu_hive_evidence` | Enable verification agent |

---

## 10. Key Source Files Reference

### Agent Tool

| File | Purpose |
|------|---------|
| `tools/AgentTool/AgentTool.tsx` | Main tool definition, input/output schemas, call() implementation |
| `tools/AgentTool/runAgent.ts` | Core agent execution loop, MCP server init, transcript recording |
| `tools/AgentTool/prompt.ts` | Agent tool prompt generation with agent listings and usage examples |
| `tools/AgentTool/constants.ts` | Tool name constants (`Agent`, `Task`), one-shot agent types |
| `tools/AgentTool/forkSubagent.ts` | Fork feature gate, fork agent definition, forked message building |
| `tools/AgentTool/agentToolUtils.ts` | Tool filtering, resolution, async lifecycle management |
| `tools/AgentTool/builtInAgents.ts` | Built-in agent registry (general-purpose, Explore, Plan, etc.) |
| `tools/AgentTool/loadAgentsDir.ts` | Custom agent loading from markdown files and settings |
| `tools/AgentTool/agentMemory.ts` | Persistent agent memory (user/project/local scopes) |
| `tools/AgentTool/agentMemorySnapshot.ts` | Memory snapshot initialization and sync |
| `tools/AgentTool/agentColorManager.ts` | Agent color assignment (8-color palette) |
| `tools/AgentTool/agentDisplay.ts` | Agent display utilities, source grouping, override resolution |
| `tools/AgentTool/resumeAgent.ts` | Resuming previously suspended agents |

### Built-In Agent Definitions

| File | Agent |
|------|-------|
| `tools/AgentTool/built-in/generalPurposeAgent.ts` | General-purpose agent (full tool access) |
| `tools/AgentTool/built-in/exploreAgent.ts` | Read-only codebase exploration |
| `tools/AgentTool/built-in/planAgent.ts` | Planning and analysis |
| `tools/AgentTool/built-in/verificationAgent.ts` | Adversarial verification specialist |
| `tools/AgentTool/built-in/claudeCodeGuideAgent.ts` | Claude Code usage guidance |
| `tools/AgentTool/built-in/statuslineSetup.ts` | Statusline configuration |

### Communication

| File | Purpose |
|------|---------|
| `tools/SendMessageTool/SendMessageTool.ts` | SendMessage tool implementation, routing, broadcast |
| `tools/SendMessageTool/prompt.ts` | SendMessage prompt with usage examples and protocol docs |
| `tools/SendMessageTool/constants.ts` | Tool name constant |

### Team Management

| File | Purpose |
|------|---------|
| `tools/TeamCreateTool/TeamCreateTool.ts` | Team creation, config file writing, task list setup |
| `tools/TeamCreateTool/prompt.ts` | TeamCreate prompt with workflow guide and best practices |
| `tools/TeamDeleteTool/TeamDeleteTool.ts` | Team cleanup, active member checks, directory removal |
| `tools/TeamDeleteTool/prompt.ts` | TeamDelete prompt |

### Coordinator

| File | Purpose |
|------|---------|
| `coordinator/coordinatorMode.ts` | Mode detection, session mode matching, system prompt, worker context |

### Swarm Infrastructure

| File | Purpose |
|------|---------|
| `utils/swarm/constants.ts` | Team lead name, tmux session names, env var constants |
| `utils/swarm/teamHelpers.ts` | Team file CRUD, member management, directory cleanup |
| `utils/swarm/inProcessRunner.ts` | In-process teammate execution loop with full lifecycle |
| `utils/swarm/spawnInProcess.ts` | In-process teammate creation, context setup, task registration |
| `utils/swarm/spawnUtils.ts` | Teammate command resolution, CLI flag propagation |
| `utils/swarm/reconnection.ts` | Teammate context restoration for fresh and resumed sessions |
| `utils/swarm/teammateInit.ts` | Teammate hooks (idle notification, team permissions) |
| `utils/swarm/teammatePromptAddendum.ts` | System prompt addendum for teammate communication |
| `utils/swarm/teammateModel.ts` | Default teammate model fallback (Opus 4.6) |
| `utils/swarm/teammateLayoutManager.ts` | Color assignment, pane creation, layout management |
| `utils/swarm/permissionSync.ts` | Cross-agent permission request/response via mailbox |
| `utils/swarm/leaderPermissionBridge.ts` | Bridge for in-process teammates to use leader's permission UI |

### Swarm Backends

| File | Purpose |
|------|---------|
| `utils/swarm/backends/types.ts` | Backend type definitions, PaneBackend interface |
| `utils/swarm/backends/registry.ts` | Backend detection, caching, registration |
| `utils/swarm/backends/InProcessBackend.ts` | In-process teammate executor |
| `utils/swarm/backends/TmuxBackend.ts` | Tmux pane management |
| `utils/swarm/backends/ITermBackend.ts` | iTerm2 pane management |
| `utils/swarm/backends/detection.ts` | Environment detection (tmux, iTerm2, it2 CLI) |
| `utils/swarm/backends/PaneBackendExecutor.ts` | Wraps PaneBackend as TeammateExecutor |
| `utils/swarm/backends/teammateModeSnapshot.ts` | Teammate mode persistence across spawns |

### Agent Identity

| File | Purpose |
|------|---------|
| `utils/teammate.ts` | Teammate identity resolution (AsyncLocalStorage + dynamic context) |
| `utils/agentSwarmsEnabled.ts` | Centralized swarm feature gate |

### Services

| File | Purpose |
|------|---------|
| `services/AgentSummary/agentSummary.ts` | Periodic 30s progress summaries via forked agent calls |
| `services/teamMemorySync/index.ts` | Team memory sync (pull/push with delta upload, OAuth, retry) |
| `services/teamMemorySync/watcher.ts` | File watcher with debounced push and failure suppression |
| `services/teamMemorySync/types.ts` | Zod schemas for team memory API responses |
| `services/teamMemorySync/secretScanner.ts` | Gitleaks-based secret detection before upload |
| `services/teamMemorySync/teamMemSecretGuard.ts` | Pre-write secret check for file tools |
