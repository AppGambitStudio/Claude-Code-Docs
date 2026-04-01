# Planning Mode, Task System, Worktrees, IDE Integration, and Context Management

## 1. Overview

Claude Code provides several interconnected systems for structured development workflows:

- **Planning Mode** transitions the session into a read-only exploration phase where Claude designs an implementation approach before writing code, requiring user approval before proceeding.
- **Git Worktree Support** creates isolated working copies of a repository so that exploratory or feature work does not interfere with the main checkout.
- **Task System** provides structured task tracking with CRUD operations, dependency management, and owner assignment -- used both for single-user sessions and multi-agent (swarm) coordination.
- **IDE Integration** connects Claude Code to editors (VS Code, JetBrains, Cursor, Windsurf) and browsers via MCP, enabling bidirectional communication.
- **Context Management** handles system context initialization, conversation compaction (auto-compact), and micro-compaction to keep conversations within model context windows.

---

## 2. Planning Mode

Planning Mode is a permission-controlled session state that restricts Claude to read-only exploration tools (Glob, Grep, Read) and prevents file writes or edits. Its purpose is to ensure alignment with the user on an implementation approach before any code changes are made.

### 2.1 Entering Plan Mode

There are two entry paths:

1. **Tool-based (automatic):** Claude calls the `EnterPlanMode` tool when it determines a task is non-trivial. The tool requires user approval -- users see a prompt and can accept or decline.

2. **Command-based (manual):** The user types `/plan` or `/plan <description>` in the CLI. If a description is provided, the session is set to plan mode and the description is submitted as a query, causing Claude to begin planning immediately.

Implementation: `tools/EnterPlanModeTool/EnterPlanModeTool.ts` and `commands/plan/plan.tsx`.

Key behaviors on entry:
- The current permission mode (e.g., `default`, `auto`) is saved as `prePlanMode` so it can be restored on exit.
- `prepareContextForPlanMode()` activates classifier side-effects when the user's default mode is `auto`.
- The tool permission context mode is set to `plan` via `applyPermissionUpdate`.
- The tool is disabled when `--channels` is active (plan mode's approval dialog requires the terminal).
- Cannot be used in agent (sub-agent) contexts.

### 2.2 Exiting Plan Mode

Exit happens through the `ExitPlanMode` tool (internally `ExitPlanModeV2Tool`). Claude calls this after writing a plan to the plan file.

Implementation: `tools/ExitPlanModeTool/ExitPlanModeV2Tool.ts`.

Key behaviors on exit:
- The tool reads the plan from the plan file on disk (not from parameters). The plan file path is session-specific.
- User approval is required for non-teammate contexts. For teammates, behavior differs:
  - If `isPlanModeRequired()` is true, the plan is sent to the team leader via mailbox for approval.
  - Otherwise, plan mode exits locally without approval.
- The previous permission mode (`prePlanMode`) is restored. If the pre-plan mode was `auto` but the auto-mode gate is now off (circuit breaker), mode falls back to `default`.
- Dangerous permissions that were stripped during plan mode are restored (unless restoring to auto mode).
- The `allowedPrompts` input field lets Claude request semantic permissions (e.g., "run tests", "install dependencies") as part of the plan exit.
- Plan content can be edited by the user through the CCR web UI or `Ctrl+G`, in which case `planWasEdited` is set to true and the edited plan is synced to disk.
- After approval, the tool result includes the plan text and the file path. If agent swarms are enabled and the Agent tool is available, a hint about `TeamCreateTool` for parallelization is included.

### 2.3 Plan Mode Permissions

In plan mode, Claude is instructed:
- DO NOT write or edit any files except the plan file.
- Explore the codebase using Glob, Grep, and Read tools.
- Use `AskUserQuestion` to clarify approaches.
- Exit plan mode with `ExitPlanMode` when ready to implement.

If the "interview phase" variant is enabled (`isPlanModeInterviewPhaseEnabled()`), detailed workflow instructions arrive via a plan_mode attachment in the messages rather than inline in the tool result.

### 2.4 Plan Mode Prompt Variants

The `EnterPlanMode` prompt has two variants selected by `USER_TYPE`:

- **External users:** Biased toward proactive planning. Claude is told to prefer using `EnterPlanMode` for implementation tasks unless they are simple. Includes extensive examples of when to plan vs. not.
- **Anthropic internal (`ant`):** More conservative. Planning is reserved for genuine ambiguity about the right approach. Simple multi-file tasks should proceed without planning. When in doubt, start working and use `AskUserQuestion` for specific questions.

### 2.5 /plan Command

The `/plan` slash command (`commands/plan/`) serves dual purposes:
- **If not in plan mode:** Enters plan mode. If arguments are provided (and not "open"), they are treated as a task description and submitted as a query.
- **If already in plan mode:** Displays the current plan content and file path. Supports `/plan open` to open the plan file in an external editor.

---

## 3. Git Worktree Support

Worktree mode provides VCS-level isolation by creating a separate working copy of the repository.

### 3.1 Feature Gate

Worktree mode is **unconditionally enabled** for all users. The function `isWorktreeModeEnabled()` in `utils/worktreeModeEnabled.ts` always returns `true`. Previously gated by a GrowthBook flag, this was removed because the cached-may-be-stale pattern was causing `--worktree` to be silently swallowed on first launch.

### 3.2 Worktree Creation (EnterWorktree)

Implementation: `tools/EnterWorktreeTool/EnterWorktreeTool.ts`.

The `EnterWorktree` tool:
- Creates an isolated git worktree inside `.claude/worktrees/` with a new branch based on HEAD.
- Alternatively, delegates to `WorktreeCreate`/`WorktreeRemove` hooks for VCS-agnostic isolation when not in a git repository.
- Accepts an optional `name` parameter. Names are validated by `validateWorktreeSlug()`: each `/`-separated segment may contain only letters, digits, dots, underscores, and dashes; max 64 chars total. If omitted, a name is generated from the plan slug or randomly.
- On creation:
  - The session CWD is changed to the worktree path via `process.chdir()` and `setCwd()`.
  - `originalCwd` is recorded via `setOriginalCwd()` for later restoration.
  - Worktree state is persisted via `saveWorktreeState()`.
  - System prompt section caches, memory file caches, and plan directory caches are cleared so they recompute for the new CWD context.
- Prevents double-entry: throws if `getCurrentWorktreeSession()` is already active.
- If invoked from within an existing worktree, resolves to the canonical git root first so `git worktree add` works correctly.

Prompt guidance: The tool should **only** be used when the user explicitly mentions "worktree". Branch operations, feature work, and bug fixes should use normal git workflow unless the user specifically requests worktree isolation.

### 3.3 Worktree Exit (ExitWorktree)

Implementation: `tools/ExitWorktreeTool/ExitWorktreeTool.ts`.

The `ExitWorktree` tool supports two actions:

| Action | Behavior |
|--------|----------|
| `keep` | Leaves the worktree directory and branch intact on disk. Tmux session (if any) continues running. |
| `remove` | Deletes the worktree directory and its branch. Kills any attached tmux session. |

Safety guards for `remove`:
- If the worktree has uncommitted files or unmerged commits, the tool **refuses** unless `discard_changes: true` is explicitly set.
- Change detection uses `git status --porcelain` and `git rev-list --count <originalHead>..HEAD`.
- If git state cannot be reliably determined (non-zero exit, missing baseline), the tool refuses (fail-closed).

Scope constraint: `ExitWorktree` only operates on worktrees created by `EnterWorktree` in the **current session**. It will not touch manually created worktrees or worktrees from previous sessions. If no active session exists, it is a no-op.

On exit:
- CWD is restored to the original directory.
- `projectRoot` is restored if it was pointed at the worktree (happens with `--worktree` startup).
- All CWD-dependent caches are cleared (system prompt sections, memory files, plans directory, hooks config snapshot).

### 3.4 Isolation Model

The worktree provides complete filesystem isolation:
- A separate `.claude/worktrees/<name>/` directory with its own git branch.
- The session switches its working directory, so all tool operations (Bash, Read, Edit, Write) operate on the worktree copy.
- On session exit (if still in the worktree), the user is prompted to keep or remove it.

---

## 4. Task System

The task system provides structured tracking for work items within a session. It replaces the older `TodoWrite` approach with a file-per-task model that supports concurrent access from multiple agents.

### 4.1 Task Model and Lifecycle

Defined in `Task.ts` and `utils/tasks.ts`.

**Task types** (from `Task.ts`):
- `local_bash` -- shell command tasks
- `local_agent` -- sub-agent tasks
- `remote_agent` -- remote agent tasks
- `in_process_teammate` -- teammate tasks running in-process
- `local_workflow` -- workflow script tasks
- `monitor_mcp` -- MCP monitor tasks
- `dream` -- dream tasks

**Task statuses** (background tasks in `Task.ts`):
- `pending` -> `running` -> `completed` | `failed` | `killed`

**Todo statuses** (task list items in `utils/tasks.ts`):
- `pending` -> `in_progress` -> `completed`

A task can also be set to `deleted`, which permanently removes its file.

**Task state fields** (`TaskStateBase`):
- `id`: Prefixed by type (e.g., `b` for bash, `a` for agent, `t` for teammate). Uses a case-insensitive-safe alphabet with 8 random characters (~2.8 trillion combinations).
- `type`, `status`, `description`, `toolUseId`
- `startTime`, `endTime`, `totalPausedMs`
- `outputFile`: Path returned by `getTaskOutputPath(id)`
- `outputOffset`, `notified`

**Todo task fields** (`Task` in `utils/tasks.ts`):
- `id`, `subject`, `description`, `activeForm` (present-continuous for spinner display)
- `owner` (agent name), `status`
- `blocks` / `blockedBy` (arrays of task IDs for dependency tracking)
- `metadata` (arbitrary key-value store)

### 4.2 Task CRUD Operations

Four tools manage the task list:

#### TaskCreate (`tools/TaskCreateTool/`)

Creates a new task with:
- `subject` (brief title), `description` (what needs to be done)
- Optional `activeForm` (e.g., "Running tests" for spinner display)
- Optional `metadata` (arbitrary key-value pairs)

Tasks are created with status `pending` and no owner. The tool auto-expands the task list view in the UI. After creation, `TaskCreated` hooks are executed; if any return a blocking error, the task is deleted and the error is thrown.

The tool is gated by `isTodoV2Enabled()`.

#### TaskUpdate (`tools/TaskUpdateTool/`)

Updates an existing task. Updatable fields:
- `subject`, `description`, `activeForm`, `status`, `owner`, `metadata`
- `addBlocks` / `addBlockedBy` (arrays of task IDs to add as dependencies)

Special behaviors:
- Setting `status: "deleted"` permanently removes the task file.
- When a teammate marks a task `in_progress` without specifying an owner, the agent name is auto-assigned.
- On ownership change with swarms enabled, the new owner receives a `task_assignment` mailbox notification.
- `TaskCompleted` hooks are executed when status transitions to `completed`; blocking errors prevent the completion.
- A structural verification nudge fires when the main-thread agent closes out 3+ tasks and none was a verification step.

#### TaskGet (`tools/TaskGetTool/`)

Retrieves a single task by ID. Returns full details including `subject`, `description`, `status`, `blocks`, and `blockedBy`. Read-only.

#### TaskList (`tools/TaskListTool/`)

Lists all tasks (excluding those with `_internal` metadata). Returns summary info: `id`, `subject`, `status`, `owner`, `blockedBy`. Completed tasks are filtered out of `blockedBy` arrays so consumers see only active blockers. Read-only.

When agent swarms are enabled, the prompt includes a teammate workflow: after completing a task, call `TaskList`, look for pending/unowned/unblocked tasks, prefer lowest ID first, and claim with `TaskUpdate`.

#### TaskStop (`tools/TaskStopTool/`)

Stops a running background task by ID. Validates that the task exists and is in `running` status before proceeding. Supports the deprecated `shell_id` parameter for backward compatibility with the old `KillShell` tool (kept as an alias).

#### TaskOutput (`tools/TaskOutputTool/`)

Reads output from a background task's output file. Registered as `TaskOutput` and supports streaming reads with offset tracking.

### 4.3 Task Dependencies

Dependencies are managed via `blocks` and `blockedBy` arrays on each task:

- `blocks`: Task IDs that cannot start until this task completes.
- `blockedBy`: Task IDs that must complete before this task can start.

Setting up dependencies: Use `TaskUpdate` with `addBlocks` or `addBlockedBy` after creating tasks. The `blockTask(taskListId, blockerId, blockedId)` function in `utils/tasks.ts` maintains bidirectional consistency.

In `TaskList` output, completed task IDs are filtered from `blockedBy` arrays so agents see only active blockers.

### 4.4 Task Storage

Tasks are stored as individual JSON files in a task list directory. The directory is determined by `getTaskListId()`:
- For team leaders: stored under the team name (matching where teammates look).
- For teammates: stored under the team name from `getTeamName()`.
- For solo sessions: stored under the session ID.

A high-water-mark file (`.highwatermark`) tracks the maximum task ID ever assigned. File locking (with async retry/backoff) is used to handle concurrent access from multiple agents in a swarm.

---

## 5. IDE Integration

### 5.1 IDE Communication Protocol

Claude Code connects to IDEs via MCP (Model Context Protocol) over either SSE or WebSocket transports. The `useIDEIntegration` hook in `hooks/useIDEIntegration.tsx` initializes the connection.

Auto-connect is enabled when any of:
- `globalConfig.autoConnectIde` is true
- The `--auto-connect-ide` CLI flag is set
- The terminal is a supported IDE terminal (`isSupportedTerminal()`)
- `CLAUDE_CODE_SSE_PORT` env var is set (supports tmux/screen that overwrite `TERM_PROGRAM`)
- `CLAUDE_CODE_AUTO_CONNECT_IDE` env var is truthy
- An IDE extension install was requested

The connection type is determined from the URL scheme:
- `ws:` prefix -> `ws-ide` (WebSocket)
- Otherwise -> `sse-ide` (Server-Sent Events)

The MCP config is registered as a dynamic config entry with scope `"dynamic"`, including `url`, `ideName`, `authToken`, and `ideRunningInWindows` fields.

### 5.2 VS Code / JetBrains Integration

Supported IDE types (from `utils/ide.ts`):
- **VS Code family:** `vscode`, `cursor`, `windsurf`
- **JetBrains family:** `pycharm`, `intellij`, `webstorm`, `phpstorm`, `rubymine`, `clion`, `goland`, `rider`, `datagrip`, `appcode`, `dataspell`, `aqua`, `gateway`, `fleet`, `androidstudio`

Integration provides:
- Bidirectional communication between Claude Code CLI and the IDE extension.
- The IDE extension install flow is managed by `initializeIdeIntegration()`, which detects the IDE, triggers onboarding, and reports installation status.
- File operations and diagnostics can be exchanged through the MCP channel.

### 5.3 Chrome Extension (Claude in Chrome)

The Chrome extension (`utils/claudeInChrome/`) provides browser automation capabilities through an MCP server.

**Setup** (`utils/claudeInChrome/setup.ts`):
- Gated by `shouldEnableClaudeInChrome()`, which checks: CLI flags, `CLAUDE_CODE_ENABLE_CFC` env var, global config `claudeInChromeDefaultEnabled`, and whether the session is interactive.
- Auto-enable logic (`shouldAutoEnableClaudeInChrome`) activates when the extension is installed and certain feature flags are met.
- The MCP server runs as a stdio subprocess (`--claude-in-chrome-mcp` argument to the same binary).
- A native host wrapper script is created in `~/.claude/chrome/` and registered with all detected Chromium browsers via native messaging host manifests.

**Supported browsers** (`utils/claudeInChrome/common.ts`):
- Chrome, Brave, Arc, Edge, Chromium, Vivaldi, Opera
- Detection order prioritizes Chrome first.
- Cross-platform: macOS (app bundle check), Linux/WSL (binary lookup), Windows (data path + registry).

**Native messaging:** Chrome's native host manifest `path` field cannot contain arguments, so a wrapper script (`chrome-native-host` / `chrome-native-host.bat`) is generated. The manifest is installed to each browser's `NativeMessagingHosts` directory. On Windows, registry entries are added.

**System prompt** (`utils/claudeInChrome/prompt.ts`):
- Provides instructions for GIF recording, console log debugging, alert avoidance, tab context management, and avoiding rabbit holes.
- When tool search is enabled, instructions to load chrome tools via `ToolSearch` before use are injected.

**Socket communication:** On Unix, per-PID sockets in `/tmp/claude-mcp-browser-bridge-<username>/`. On Windows, named pipes (`\\.\pipe\claude-mcp-browser-bridge-<username>`). Legacy fallback paths are also checked.

The `/chrome` command (`commands/chrome/`) provides Claude in Chrome settings, available only in interactive sessions (availability: `claude-ai`).

### 5.4 Desktop App Integration

The `/desktop` command (`commands/desktop/`) enables handing off the current session to Claude Desktop. It renders a `DesktopHandoff` component.

- **Supported platforms:** macOS (any arch), Windows x64.
- Aliased as `/app`.
- Availability restricted to `claude-ai`.
- Hidden on unsupported platforms.

---

## 6. Context Management

### 6.1 Context Initialization

System context is built and memoized in `context.ts`. Key components:

- **Git status snapshot:** Captured at conversation start (branch, main branch, short status, recent commits, user name). Truncated to 2000 chars if needed. Skipped in CCR (remote) sessions or when git instructions are disabled.
- **CLAUDE.md files:** Memory files loaded from the project, home directory, and additional configured directories via `getClaudeMds()` and `getMemoryFiles()`.
- **Cache breaker injection:** An ant-only mechanism (`systemPromptInjection`) that can inject arbitrary strings into the system prompt to break the prompt cache for debugging.
- **User context:** Built from memory files and other session-specific information.

Both `getSystemContext()` and `getUserContext()` are memoized for the duration of the conversation and cleared when injection changes or during compaction.

### 6.2 Message Compression (Compaction)

Compaction is the primary mechanism for keeping conversations within model context windows. Implemented in `services/compact/`.

**Auto-compact** (`services/compact/autoCompact.ts`):
- Triggers when estimated token count exceeds the effective context window size minus a buffer.
- The effective window is: `contextWindow - max(maxOutputTokens, 20000)`.
- Buffer: `AUTOCOMPACT_BUFFER_TOKENS = 13,000` tokens.
- A circuit breaker stops retrying after `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3` consecutive failures.
- The threshold can be overridden via `CLAUDE_CODE_AUTO_COMPACT_WINDOW` or `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` env vars.

**Full compaction** (`services/compact/compact.ts`):
- Uses a forked agent call to summarize the conversation into a compact form.
- The summary prompt (`services/compact/prompt.ts`) instructs the model to produce a detailed analysis in `<analysis>` tags (stripped before use) followed by structured sections: primary request/intent, key technical concepts, files and code sections, errors and fixes, problem solving, all user messages, pending tasks, current work, and optional next step.
- Supports partial compaction: only recent messages after a compact boundary are summarized, while older context is preserved.
- Post-compaction cleanup runs hooks, re-appends session metadata, and notifies the prompt cache system.

**Micro-compaction** (`services/compact/microCompact.ts`):
- A lighter-weight approach that compacts individual tool results in-place without a full conversation summary.
- Targets specific tools: `Read`, `Bash`, `Grep`, `Glob`, `WebSearch`, `WebFetch`, `Edit`, `Write`.
- Uses a time-based strategy: older tool results are cleared first (`[Old tool result content cleared]`).
- Images are estimated at `IMAGE_MAX_TOKEN_SIZE = 2000` tokens.
- A cached micro-compact variant (ant-only, gated by feature flag) maintains state for incremental compaction.

**Compact prompt structure:**
- A `NO_TOOLS_PREAMBLE` is prepended to prevent the compaction model from calling tools.
- Two instruction variants: `BASE` (full conversation) and `PARTIAL` (recent messages only).
- The summary covers 9 sections with emphasis on verbatim quotes, file names, code snippets, and user feedback.

### 6.3 Context Truncation

Context can be truncated at compact boundary messages. The `getMessagesAfterCompactBoundary()` utility retrieves only messages after the last compaction boundary, while `createCompactBoundaryMessage()` inserts the boundary marker. This allows the conversation to continue with the summary as the new starting context, discarding older messages.

Warning thresholds:
- `WARNING_THRESHOLD_BUFFER_TOKENS = 20,000` -- shows a warning to the user.
- `ERROR_THRESHOLD_BUFFER_TOKENS = 20,000` -- triggers auto-compact.
- `MANUAL_COMPACT_BUFFER_TOKENS = 3,000` -- threshold for user-triggered compact.

---

## 7. Key Source Files Reference

| Area | File | Purpose |
|------|------|---------|
| **Plan Mode** | `tools/EnterPlanModeTool/EnterPlanModeTool.ts` | Tool to enter plan mode (requires user approval) |
| | `tools/EnterPlanModeTool/prompt.ts` | Prompt with when-to-use guidance (external and ant variants) |
| | `tools/ExitPlanModeTool/ExitPlanModeV2Tool.ts` | Tool to exit plan mode, reads plan from disk, handles teammate approval flow |
| | `tools/ExitPlanModeTool/prompt.ts` | Prompt explaining plan file workflow and when to exit |
| | `commands/plan/plan.tsx` | `/plan` slash command: enters plan mode or displays current plan |
| | `commands/plan/index.ts` | Command registration for `/plan` |
| **Worktree** | `tools/EnterWorktreeTool/EnterWorktreeTool.ts` | Creates git worktree and switches session CWD |
| | `tools/EnterWorktreeTool/prompt.ts` | Prompt: only use when user explicitly says "worktree" |
| | `tools/ExitWorktreeTool/ExitWorktreeTool.ts` | Exits worktree (keep or remove), restores CWD, safety checks |
| | `tools/ExitWorktreeTool/prompt.ts` | Prompt: scope, parameters, and cleanup behavior |
| | `utils/worktreeModeEnabled.ts` | Feature gate (unconditionally enabled) |
| **Task System** | `Task.ts` | Task type/status definitions, ID generation, base state factory |
| | `tasks.ts` | Task registry: `getAllTasks()`, `getTaskByType()` |
| | `utils/tasks.ts` | File-based task storage: CRUD, locking, signals, schema |
| | `tools/TaskCreateTool/TaskCreateTool.ts` | Create task with subject, description, hooks |
| | `tools/TaskUpdateTool/TaskUpdateTool.ts` | Update task fields, status transitions, dependency management |
| | `tools/TaskGetTool/TaskGetTool.ts` | Retrieve single task by ID |
| | `tools/TaskListTool/TaskListTool.ts` | List all tasks with summary info |
| | `tools/TaskStopTool/TaskStopTool.ts` | Stop a running background task |
| | `tools/TaskOutputTool/TaskOutputTool.tsx` | Read output from background task |
| **IDE** | `hooks/useIDEIntegration.tsx` | React hook: auto-detect and connect to IDE via MCP |
| | `utils/ide.ts` | IDE type definitions, detection, terminal support |
| **Chrome** | `utils/claudeInChrome/setup.ts` | Chrome extension MCP server setup, native host installation |
| | `utils/claudeInChrome/common.ts` | Browser configs, detection, socket paths, URL opening |
| | `utils/claudeInChrome/prompt.ts` | System prompt for browser automation |
| | `commands/chrome/index.ts` | `/chrome` command registration |
| **Desktop** | `commands/desktop/desktop.tsx` | `/desktop` command: hand off session to Claude Desktop |
| | `commands/desktop/index.ts` | Command registration, platform check (macOS, Windows x64) |
| **Context** | `context.ts` | System/user context initialization, git status, memoization |
| | `services/compact/compact.ts` | Full conversation compaction via forked agent |
| | `services/compact/autoCompact.ts` | Auto-compact trigger logic, thresholds, circuit breaker |
| | `services/compact/microCompact.ts` | In-place tool result compaction, time-based clearing |
| | `services/compact/prompt.ts` | Compaction summary prompt (analysis + 9 sections) |
