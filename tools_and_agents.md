# Tools and Agent Ecosystem

Claude Code's tool system is the core mechanism through which the AI interacts with the host machine. Every file read, shell command, web search, and sub-agent spawn goes through a unified tool architecture defined in `Tool.ts`, registered in `tools.ts`, and gated by a layered permission system in `hooks/useCanUseTool.tsx`.

---

## 1. Tool Architecture

### The `Tool` Type

Every tool in the system conforms to the `Tool<Input, Output, P>` generic type defined in `Tool.ts`. The three type parameters are:

- **`Input`** -- a Zod schema (`z.ZodType`) describing the tool's input object
- **`Output`** -- the type of the data returned from the tool's `call()` method
- **`P`** -- a `ToolProgressData` subtype for streaming progress events during execution

```ts
export type Tool<
  Input extends AnyObject = AnyObject,
  Output = unknown,
  P extends ToolProgressData = ToolProgressData,
> = {
  readonly name: string
  aliases?: string[]
  searchHint?: string          // 3-10 word phrase for ToolSearch keyword matching
  readonly inputSchema: Input
  readonly inputJSONSchema?: ToolInputJSONSchema  // Alternative JSON Schema (used by MCP tools)
  outputSchema?: z.ZodType<unknown>
  maxResultSizeChars: number   // Threshold before result is persisted to disk
  readonly shouldDefer?: boolean
  readonly alwaysLoad?: boolean
  readonly strict?: boolean    // Enables stricter API adherence to schema
  isMcp?: boolean
  isLsp?: boolean

  // Lifecycle methods (see below)
  validateInput?(input, context): Promise<ValidationResult>
  checkPermissions(input, context): Promise<PermissionResult>
  call(args, context, canUseTool, parentMessage, onProgress?): Promise<ToolResult<Output>>

  // Metadata methods
  description(input, options): Promise<string>
  prompt(options): Promise<string>
  userFacingName(input): string
  isEnabled(): boolean
  isConcurrencySafe(input): boolean
  isReadOnly(input): boolean
  isDestructive?(input): boolean
  isSearchOrReadCommand?(input): { isSearch; isRead; isList? }
  interruptBehavior?(): 'cancel' | 'block'

  // Rendering methods
  renderToolUseMessage(input, options): React.ReactNode
  renderToolResultMessage?(content, progressMessages, options): React.ReactNode
  renderToolUseProgressMessage?(progressMessages, options): React.ReactNode
  renderToolUseErrorMessage?(result, options): React.ReactNode
  renderToolUseRejectedMessage?(input, options): React.ReactNode
  renderGroupedToolUse?(toolUses, options): React.ReactNode | null
  mapToolResultToToolResultBlockParam(content, toolUseID): ToolResultBlockParam

  // ...additional optional methods
}
```

### `ToolDef` and `buildTool()`

Individual tool files do not implement `Tool` directly. They implement `ToolDef`, which makes several commonly-stubbed methods optional, then pass the definition to `buildTool()`:

```ts
export type ToolDef<Input, Output, P> =
  Omit<Tool<Input, Output, P>, DefaultableToolKeys> &
  Partial<Pick<Tool<Input, Output, P>, DefaultableToolKeys>>
```

The defaultable keys and their defaults (fail-closed where it matters):

| Key | Default |
|---|---|
| `isEnabled` | `() => true` |
| `isConcurrencySafe` | `() => false` (assume not safe) |
| `isReadOnly` | `() => false` (assume writes) |
| `isDestructive` | `() => false` |
| `checkPermissions` | `{ behavior: 'allow', updatedInput }` (defer to general permission system) |
| `toAutoClassifierInput` | `() => ''` (skip classifier) |
| `userFacingName` | `() => name` |

`buildTool<D>(def: D)` produces a `BuiltTool<D>` via `{ ...TOOL_DEFAULTS, userFacingName: () => def.name, ...def }`, so the caller's overrides always win.

### `ToolResult<T>`

Every tool's `call()` returns a `ToolResult<T>`:

```ts
export type ToolResult<T> = {
  data: T
  newMessages?: (UserMessage | AssistantMessage | AttachmentMessage | SystemMessage)[]
  contextModifier?: (context: ToolUseContext) => ToolUseContext
  mcpMeta?: { _meta?; structuredContent? }
}
```

The `data` field is the actual output. `newMessages` allows a tool to inject synthetic messages into the conversation. `contextModifier` lets non-concurrency-safe tools mutate the context for subsequent calls in the same turn.

### Tool Lifecycle

When the model emits a `tool_use` block, the system executes these steps in order:

1. **Input parsing** -- The raw JSON is validated against the tool's `inputSchema` (Zod).
2. **`validateInput()`** -- Tool-specific input validation (e.g., BashTool checks command structure, FileEditTool verifies file size). Returns `{ result: true }` or `{ result: false, message, errorCode }`.
3. **Permission check** -- The `useCanUseTool` hook orchestrates a multi-stage decision:
   - `hasPermissionsToUseTool()` evaluates deny/allow/ask rules from settings
   - `checkPermissions()` on the tool runs tool-specific permission logic
   - If the result is `ask`, the interactive or coordinator handler prompts the user
4. **`call()`** -- The tool executes its core logic, optionally streaming progress via `onProgress`.
5. **Result mapping** -- `mapToolResultToToolResultBlockParam()` converts the output into the API's `ToolResultBlockParam` format for the next model turn.
6. **Result persistence** -- If the result exceeds `maxResultSizeChars`, it is saved to disk and the model receives a preview with a file path reference.

### `ToolUseContext`

The `ToolUseContext` object passed to every tool carries the full execution environment:

```ts
export type ToolUseContext = {
  options: {
    commands: Command[]
    tools: Tools
    mcpClients: MCPServerConnection[]
    mainLoopModel: string
    thinkingConfig: ThinkingConfig
    isNonInteractiveSession: boolean
    agentDefinitions: AgentDefinitionsResult
    // ...
  }
  abortController: AbortController
  readFileState: FileStateCache
  getAppState(): AppState
  setAppState(f: (prev: AppState) => AppState): void
  messages: Message[]
  // ...many more fields for subagent tracking, file history, attribution, etc.
}
```

### Tool Registration

All tools are registered in `tools.ts` through the `getAllBaseTools()` function. This is the single source of truth for available tools. The function returns a `Tools` (which is `readonly Tool[]`) array containing:

- Unconditionally included tools: `AgentTool`, `BashTool`, `FileReadTool`, `FileEditTool`, `FileWriteTool`, `NotebookEditTool`, `WebFetchTool`, `WebSearchTool`, `TodoWriteTool`, `TaskStopTool`, `TaskOutputTool`, `AskUserQuestionTool`, `SkillTool`, `EnterPlanModeTool`, `ExitPlanModeV2Tool`, `SendMessageTool`, `BriefTool`
- Conditionally included based on feature flags or environment: `GlobTool`/`GrepTool` (omitted when embedded search tools exist), `LSPTool`, `TaskCreateTool`/`TaskGetTool`/`TaskUpdateTool`/`TaskListTool`, `EnterWorktreeTool`/`ExitWorktreeTool`, `ToolSearchTool`, `SleepTool`, `REPLTool`, `PowerShellTool`, and others

The `getTools(permissionContext)` function filters `getAllBaseTools()` by:
1. Removing tools that match blanket deny rules in the permission context
2. Hiding primitive tools when REPL mode wraps them
3. Filtering out disabled tools via `isEnabled()`

The `assembleToolPool(permissionContext, mcpTools)` function combines built-in tools with MCP tools, deduplicating by name (built-ins win on conflict) and sorting each partition for prompt-cache stability.

---

## 2. File System Tools

### BashTool

**Source:** `tools/BashTool/BashTool.tsx` (plus ~15 supporting files)

The BashTool dispatches commands to a sub-shell and captures output. It is the most complex tool in the system.

**Input schema:**
```ts
z.object({
  command: z.string(),          // The shell command to execute
  timeout: z.number().optional(), // Optional timeout in milliseconds
  description: z.string().optional(),
  run_in_background: z.boolean().optional(),
  dangerouslyDisableSandbox: z.boolean().optional(),
})
```

**Key behaviors:**
- **Shell execution** via `exec()` from `utils/Shell.ts`, capturing stdout/stderr into an `EndTruncatingAccumulator`
- **Sandboxing** -- `shouldUseSandbox.ts` determines whether a command runs in a sandbox (via `SandboxManager` from `utils/sandbox/sandbox-adapter.ts`). Users can override with `dangerouslyDisableSandbox`
- **Background execution** -- When `run_in_background` is true, the command spawns as a `LocalShellTask` and the tool returns immediately with a task ID. Progress is reported asynchronously
- **Auto-backgrounding** -- In assistant mode, blocking commands auto-background after `ASSISTANT_BLOCKING_BUDGET_MS` (15 seconds)
- **Command classification** -- `isSearchOrReadBashCommand()` categorizes commands for collapsible UI display. `BASH_SEARCH_COMMANDS` (find, grep, rg, ag...), `BASH_READ_COMMANDS` (cat, head, jq, awk...), `BASH_LIST_COMMANDS` (ls, tree, du), and `BASH_SEMANTIC_NEUTRAL_COMMANDS` (echo, printf) are classified
- **Security** -- `bashSecurity.ts` and `bashPermissions.ts` implement command-level permission checking with wildcard pattern matching. `readOnlyValidation.ts` enforces read-only constraints. `sedValidation.ts` and `sedEditParser.ts` detect `sed` edit operations for file-edit tracking
- **File tracking** -- Git operation tracking (`trackGitOperations`), file history tracking (`fileHistoryTrackEdit`), and VS Code notification (`notifyVscodeFileUpdated`) happen post-execution
- **Image output** -- `buildImageToolResult()` and `resizeShellImageOutput()` handle commands that produce image output

**Permission model:** Tool-specific permission logic lives in `bashPermissions.ts`. The `bashToolHasPermission()` function evaluates command content against allow/deny rules with wildcard pattern matching (e.g., `"git *"` matches any git command). A bash classifier can speculatively approve commands in auto mode.

### FileEditTool

**Source:** `tools/FileEditTool/FileEditTool.ts`

Performs targeted string replacements in files. Does not rewrite entire files.

**Input schema** (from `types.ts`):
```ts
z.strictObject({
  file_path: z.string(),
  old_string: z.string(),
  new_string: z.string(),
  replace_all: z.boolean().default(false),
})
```

**Key behaviors:**
- Uses `findActualString()` and `getPatchForEdit()` from `utils.ts` to locate the exact match and compute a minimal patch
- Validates that `old_string` is unique in the file (unless `replace_all` is true)
- Tracks file modifications with `fileHistoryTrackEdit` and notifies VS Code
- Integrates with LSP diagnostic tracking (`clearDeliveredDiagnosticsForFile`)
- Enforces a 1 GiB file size limit (`MAX_EDIT_FILE_SIZE`)
- Marked as `strict: true` for stricter API schema adherence

### FileReadTool

**Source:** `tools/FileReadTool/FileReadTool.ts`

Reads file contents with support for line ranges, images, PDFs, and Jupyter notebooks. Input accepts `file_path`, optional `offset`/`limit` for line ranges, and optional `pages` for PDF page ranges.

**Key behaviors:**
- Handles binary files (images via `imageProcessor.ts`, PDFs via `utils/pdf.ts`)
- Reads Jupyter notebooks via `utils/notebook.ts`, returning cells with outputs
- Applies token counting and limits from `limits.ts` (`getDefaultFileReadingLimits`)
- `maxResultSizeChars` is `Infinity` since the tool self-bounds via its own limits

### FileWriteTool

**Source:** `tools/FileWriteTool/FileWriteTool.ts`

Creates new files or completely overwrites existing ones.

**Input schema:**
```ts
z.strictObject({
  file_path: z.string(),
  content: z.string(),
})
```

Returns diff information including `structuredPatch` (array of hunks), `originalFile`, and the new content. Integrates with file history tracking and permission checks via `checkWritePermissionForTool`.

### GlobTool

**Source:** `tools/GlobTool/GlobTool.ts`

Fast file pattern matching using `utils/glob.ts`. Supports glob patterns like `**/*.ts`.

**Input schema:**
```ts
z.strictObject({
  pattern: z.string(),
  path: z.string().optional(),
})
```

Returns `{ durationMs, numFiles, filenames, truncated }`. Results are capped at a configurable limit. Marked as `isConcurrencySafe: true` and `isReadOnly: true`.

### GrepTool

**Source:** `tools/GrepTool/GrepTool.ts`

Content search powered by ripgrep (`utils/ripgrep.ts`). Supports full regex syntax, output modes, context lines, and file type filtering.

**Input schema** (selected fields):
```ts
z.strictObject({
  pattern: z.string(),
  path: z.string().optional(),
  glob: z.string().optional(),
  output_mode: z.enum(['content', 'files_with_matches', 'count']).optional(),
  '-B': z.number().optional(),  // Lines before match
  '-A': z.number().optional(),  // Lines after match
  '-C': z.number().optional(),  // Context lines
  '-n': z.boolean().optional(), // Line numbers
  '-i': z.boolean().optional(), // Case insensitive
  type: z.string().optional(),  // File type filter (rg --type)
  head_limit: z.number().optional(),
  offset: z.number().optional(),
  multiline: z.boolean().optional(),
})
```

Both GlobTool and GrepTool are conditionally excluded when the build has embedded search tools (`hasEmbeddedSearchTools()` -- ant-native builds embed bfs/ugrep in the binary).

---

## 3. Web Tools

### WebSearchTool

**Source:** `tools/WebSearchTool/WebSearchTool.ts`

Performs web searches using the Anthropic API's built-in web search capability (`BetaWebSearchTool20250305`).

**Input schema:**
```ts
z.strictObject({
  query: z.string().min(2),
  allowed_domains: z.array(z.string()).optional(),
  blocked_domains: z.array(z.string()).optional(),
})
```

Internally creates a sub-request to the model with `web_search_20250305` as a tool, then extracts and returns the search results. Uses a smaller/faster model (`getSmallFastModel`) for the search sub-request to reduce latency and cost.

### WebFetchTool

**Source:** `tools/WebFetchTool/WebFetchTool.ts`

Fetches content from a URL, converts it to markdown, and applies a prompt to extract relevant information.

**Input schema:**
```ts
z.strictObject({
  url: z.string().url(),
  prompt: z.string(),
})
```

**Key behaviors:**
- `getURLMarkdownContent()` fetches and converts HTML to markdown
- `applyPromptToMarkdown()` uses the model to extract relevant content
- Pre-approved hosts skip permission prompts (`isPreapprovedUrl()`, `isPreapprovedHost()`)
- Permission rules use `domain:hostname` format for allow/deny rules
- Marked as `shouldDefer: true` (loaded via ToolSearch when many tools are present)
- Returns `{ bytes, code, codeText, result, durationMs, url }`

---

## 4. Protocol Tools

### MCPTool

**Source:** `tools/MCPTool/MCPTool.ts`

A template tool for Model Context Protocol actions. The actual MCP tools are dynamically created by cloning and overriding this template in `services/mcp/mcpClient.ts`. Each connected MCP server produces tool instances with real `name`, `description`, `call`, and `inputSchema` values.

```ts
export const MCPTool = buildTool({
  isMcp: true,
  name: 'mcp',                  // Overridden per-instance
  async call() { return { data: '' } },  // Overridden per-instance
  async checkPermissions() { return { behavior: 'passthrough' } },
  // ...
} satisfies ToolDef<InputSchema, Output>)
```

The passthrough input schema (`z.object({}).passthrough()`) accepts any object since MCP tools define their own schemas. The `mcpInfo` field on each instance carries the original `{ serverName, toolName }` pair.

MCP tools support `shouldDefer` (sent with `defer_loading: true`) and `alwaysLoad` (via `_meta['anthropic/alwaysLoad']`) to control whether their full schema appears in the initial prompt or requires a ToolSearch round-trip.

### LSPTool

**Source:** `tools/LSPTool/LSPTool.ts`

Connects to Language Server Protocol servers for code intelligence operations.

**Input schema:**
```ts
z.strictObject({
  operation: z.enum([
    'goToDefinition', 'findReferences', 'hover', 'documentSymbol',
    'workspaceSymbol', 'goToImplementation', 'prepareCallHierarchy',
    'incomingCalls', 'outgoingCalls',
  ]),
  filePath: z.string(),
  line: z.number().int().positive(),
  character: z.number().int().min(0),
  query: z.string().optional(),  // For workspaceSymbol
})
```

Uses the LSP server manager (`services/lsp/manager.ts`) with `waitForInitialization()` before performing operations. Results are formatted by operation-specific formatters in `formatters.ts`. Only enabled when `ENABLE_LSP_TOOL` environment variable is set.

### NotebookEditTool

**Source:** `tools/NotebookEditTool/NotebookEditTool.ts`

Edits Jupyter notebook (`.ipynb`) cells.

**Input schema:**
```ts
z.strictObject({
  notebook_path: z.string(),
  cell_id: z.string().optional(),
  new_source: z.string(),
  cell_type: z.enum(['code', 'markdown']).optional(),
  edit_mode: z.enum(['replace', 'insert', 'delete']).optional(),
})
```

Parses and writes the notebook's JSON structure directly, supporting cell replacement, insertion, and deletion. Integrates with file history tracking.

---

## 5. Agent Tools

### AgentTool

**Source:** `tools/AgentTool/AgentTool.tsx` (plus ~12 supporting files)

The AgentTool spawns sub-agent instances that run in isolated contexts. This is how Claude Code delegates parallel or specialized work.

**Input schema** (full version with all gated fields):
```ts
z.object({
  description: z.string(),       // Short 3-5 word task description
  prompt: z.string(),            // The task for the agent to perform
  subagent_type: z.string().optional(),  // Specialized agent type
  model: z.enum(['sonnet', 'opus', 'haiku']).optional(),
  run_in_background: z.boolean().optional(),
  name: z.string().optional(),   // Addressable name for SendMessage
  team_name: z.string().optional(),
  mode: permissionModeSchema().optional(),
  isolation: z.enum(['worktree', 'remote']).optional(),
  cwd: z.string().optional(),    // Working directory override
})
```

**Key behaviors:**
- **Foreground agents** run synchronously within the parent's turn, blocking until complete. Their results are returned inline.
- **Background agents** (`run_in_background: true`) are registered via `registerAsyncAgent()` and the tool returns immediately with an `agentId`. The parent is notified on completion.
- **Auto-backgrounding** -- Foreground agents auto-background after `getAutoBackgroundMs()` (120s when enabled).
- **Isolation modes:**
  - `worktree` -- Creates a temporary git worktree (`createAgentWorktree`) so the agent operates on an isolated copy of the repo
  - `remote` -- Launches in a remote Claude Code Runner environment (ant-only)
- **Agent types** -- `subagent_type` selects from registered agent definitions (loaded from `loadAgentsDir.ts`). The default is `GENERAL_PURPOSE_AGENT`. Built-in agent types are in `built-in/`.
- **Fork subagents** -- `forkSubagent.ts` implements a cache-sharing fork mechanism where the sub-agent inherits the parent's rendered system prompt to avoid cache invalidation.
- **Memory** -- `agentMemory.ts` and `agentMemorySnapshot.ts` handle context transfer to sub-agents.
- **Tool pool** -- Sub-agents get their own tool pool assembled via `assembleToolPool()`, with certain tools filtered out (`ALL_AGENT_DISALLOWED_TOOLS`, `CUSTOM_AGENT_DISALLOWED_TOOLS`).

**Output** is a discriminated union:
- `{ status: 'completed', result, ... }` -- Synchronous completion
- `{ status: 'async_launched', agentId, description, prompt }` -- Background launch

### SendMessageTool

**Source:** `tools/SendMessageTool/SendMessageTool.ts`

Sends messages between agents in multi-agent (swarm) configurations.

**Input schema:**
```ts
z.object({
  to: z.string(),         // Recipient: teammate name, "*" for broadcast, "uds:<path>", "bridge:<id>"
  summary: z.string().optional(),
  message: z.union([z.string(), StructuredMessage]),
})
```

Supports structured message types including `shutdown_request`, `shutdown_response`, and `plan_approval_response`. Routes messages through teammate mailboxes (`writeToMailbox`), local agent task queues (`queuePendingMessage`), UDS sockets, or REPL bridges.

---

## 6. Utility Tools

### ToolSearchTool

**Source:** `tools/ToolSearchTool/ToolSearchTool.ts`

Implements deferred tool loading. When the total tool count exceeds a threshold, low-priority tools are sent to the model with `defer_loading: true` (just a name and search hint). The model must call ToolSearchTool to retrieve their full schemas before invoking them.

**Input schema:**
```ts
z.object({
  query: z.string(),        // "select:ToolName" for direct lookup, or keywords for search
  max_results: z.number().optional().default(5),
})
```

**Key behaviors:**
- `"select:Read,Edit,Grep"` -- Direct selection by exact name
- `"notebook jupyter"` -- Keyword search scored against tool descriptions
- Uses memoized tool descriptions for scoring (`getToolDescriptionMemoized`)
- Returns matched tools' full JSON schemas inside a `<functions>` block
- Checks `isToolSearchEnabledOptimistic()` to decide whether to include itself in the tool pool

### AskUserQuestionTool

**Source:** `tools/AskUserQuestionTool/AskUserQuestionTool.tsx`

Presents structured questions to the user with 2-4 selectable options, optional preview content, and multi-select support. Uses a custom React-based UI with `questionSchema` defining the question format. Supports option previews (code snippets, mockups) and per-question annotations/notes.

### SleepTool

**Source:** `tools/SleepTool/prompt.ts`

Pauses execution for a specified duration. Only available when the `PROACTIVE` or `KAIROS` feature flag is enabled. Used by proactive/background agents that need to wait for external events.

---

## 7. Task Tools

The task tools provide a structured planning/tracking system. They are enabled when `isTodoV2Enabled()` returns true.

| Tool | File | Purpose |
|---|---|---|
| `TaskCreateTool` | `tools/TaskCreateTool/` | Create a task with subject, description, and optional metadata |
| `TaskUpdateTool` | `tools/TaskUpdateTool/` | Update task status (in_progress, completed, etc.) |
| `TaskGetTool` | `tools/TaskGetTool/` | Retrieve a specific task by ID |
| `TaskListTool` | `tools/TaskListTool/` | List all tasks, optionally filtered by status |
| `TaskOutputTool` | `tools/TaskOutputTool/` | Read output from a background agent task |
| `TaskStopTool` | `tools/TaskStopTool/` | Stop/cancel a running background task |

`TaskCreateTool` executes task-created hooks and returns the created task object with an ID. Task data is managed through `utils/tasks.ts`. For detailed planning workflows, see the planning documentation.

---

## 8. Mode Tools

These tools transition the agent between operational modes.

| Tool | File | Purpose |
|---|---|---|
| `EnterPlanModeTool` | `tools/EnterPlanModeTool/` | Switch to plan mode (read-only, design-before-code). Takes no parameters. Calls `handlePlanModeTransition()` and `prepareContextForPlanMode()` |
| `ExitPlanModeV2Tool` | `tools/ExitPlanModeTool/` | Exit plan mode and resume normal execution |
| `EnterWorktreeTool` | `tools/EnterWorktreeTool/` | Create a git worktree for isolated work. Accepts an optional `name` parameter. Creates via `createWorktreeForSession()` and updates cwd |
| `ExitWorktreeTool` | `tools/ExitWorktreeTool/` | Leave the current worktree and return to the original working directory |

Worktree tools are gated behind `isWorktreeModeEnabled()`. Plan mode tools save/restore the permission mode via `ToolPermissionContext.prePlanMode`. For detailed planning workflows, see the planning documentation.

---

## 9. Tool Permission Validation

### The `useCanUseTool` Hook

**Source:** `hooks/useCanUseTool.tsx`

The `useCanUseTool` hook is the central permission gate. It returns a `CanUseToolFn` that every tool call passes through before execution.

```ts
export type CanUseToolFn<Input = Record<string, unknown>> = (
  tool: ToolType,
  input: Input,
  toolUseContext: ToolUseContext,
  assistantMessage: AssistantMessage,
  toolUseID: string,
  forceDecision?: PermissionDecision<Input>,
) => Promise<PermissionDecision<Input>>
```

### Permission Decision Flow

1. **`hasPermissionsToUseTool()`** (from `utils/permissions/permissions.ts`) evaluates the tool + input against the `ToolPermissionContext`:
   - **Deny rules** (`alwaysDenyRules`) -- Immediately reject with `{ behavior: 'deny' }`
   - **Allow rules** (`alwaysAllowRules`) -- Immediately approve with `{ behavior: 'allow' }`
   - **Ask rules** (`alwaysAskRules`) -- Force a prompt regardless of other signals
   - Falls through to `{ behavior: 'ask' }` if no rule matches

2. If the result is `allow`, the hook resolves immediately. A transcript classifier may tag the decision for analytics (`setYoloClassifierApproval`).

3. If the result is `deny`, the denial is logged. In auto mode, denials from the classifier are tracked via `recordAutoModeDenial()` and a notification is shown.

4. If the result is `ask`, multiple handlers attempt to resolve without prompting:
   - **Coordinator handler** (`handleCoordinatorPermission`) -- For coordinator workers that await automated checks before showing UI
   - **Swarm worker handler** (`handleSwarmWorkerPermission`) -- For teammate agents that can't show interactive UI
   - **Bash classifier** -- A speculative classifier check (`peekSpeculativeClassifierCheck`) may auto-approve bash commands with high confidence
   - **Interactive handler** (`handleInteractivePermission`) -- Falls back to showing the user a permission prompt UI

### `ToolPermissionContext`

```ts
export type ToolPermissionContext = DeepImmutable<{
  mode: PermissionMode           // 'default' | 'plan' | 'auto' | etc.
  additionalWorkingDirectories: Map<string, AdditionalWorkingDirectory>
  alwaysAllowRules: ToolPermissionRulesBySource
  alwaysDenyRules: ToolPermissionRulesBySource
  alwaysAskRules: ToolPermissionRulesBySource
  isBypassPermissionsModeAvailable: boolean
  isAutoModeAvailable?: boolean
  shouldAvoidPermissionPrompts?: boolean   // Background agents that can't show UI
  awaitAutomatedChecksBeforeDialog?: boolean
  prePlanMode?: PermissionMode
}>
```

Rules are keyed by source (settings file, user session, etc.) via `ToolPermissionRulesBySource`. The `filterToolsByDenyRules()` function in `tools.ts` pre-filters the tool pool before the model even sees the tools, so blanket-denied tools never appear in the prompt.

---

## 10. Key Source Files

| File | Purpose |
|---|---|
| `Tool.ts` | Core types: `Tool`, `ToolDef`, `ToolUseContext`, `ToolResult`, `ValidationResult`, `ToolPermissionContext`, `buildTool()` |
| `tools.ts` | Tool registration: `getAllBaseTools()`, `getTools()`, `assembleToolPool()`, `getMergedTools()`, `filterToolsByDenyRules()` |
| `hooks/useCanUseTool.tsx` | Permission orchestration: `CanUseToolFn`, interactive/coordinator/swarm permission handlers |
| `hooks/toolPermission/` | Permission handler implementations and context factories |
| `utils/permissions/permissions.ts` | Core permission evaluation: `hasPermissionsToUseTool()`, deny rule matching |
| `utils/permissions/filesystem.ts` | File-level permission checks: `checkReadPermissionForTool()`, `checkWritePermissionForTool()` |
| `tools/BashTool/BashTool.tsx` | Shell execution, stdout/stderr capture, sandboxing, background tasks |
| `tools/BashTool/bashPermissions.ts` | Bash command permission matching with wildcard patterns |
| `tools/BashTool/shouldUseSandbox.ts` | Sandbox decision logic |
| `tools/FileEditTool/FileEditTool.ts` | Targeted string replacement in files |
| `tools/FileReadTool/FileReadTool.ts` | File reading with image/PDF/notebook support |
| `tools/FileWriteTool/FileWriteTool.ts` | File creation and overwrite |
| `tools/GlobTool/GlobTool.ts` | Fast file pattern matching |
| `tools/GrepTool/GrepTool.ts` | Ripgrep-powered content search |
| `tools/WebSearchTool/WebSearchTool.ts` | Web search via Anthropic API |
| `tools/WebFetchTool/WebFetchTool.ts` | URL fetching with markdown conversion |
| `tools/MCPTool/MCPTool.ts` | MCP tool template (cloned per MCP server tool) |
| `tools/LSPTool/LSPTool.ts` | Language Server Protocol operations |
| `tools/AgentTool/AgentTool.tsx` | Sub-agent spawning, foreground/background/fork modes |
| `tools/AgentTool/runAgent.ts` | Agent execution loop |
| `tools/AgentTool/forkSubagent.ts` | Cache-sharing fork mechanism |
| `tools/AgentTool/loadAgentsDir.ts` | Agent definition loading and filtering |
| `tools/SendMessageTool/SendMessageTool.ts` | Inter-agent messaging |
| `tools/ToolSearchTool/ToolSearchTool.ts` | Deferred tool loading and keyword search |
| `tools/AskUserQuestionTool/AskUserQuestionTool.tsx` | Structured user questioning with options |
| `tools/NotebookEditTool/NotebookEditTool.ts` | Jupyter notebook cell editing |
| `tools/TaskCreateTool/TaskCreateTool.ts` | Task creation for planning/tracking |
| `tools/EnterPlanModeTool/EnterPlanModeTool.ts` | Plan mode entry |
| `tools/EnterWorktreeTool/EnterWorktreeTool.ts` | Git worktree creation |
| `constants/tools.ts` | Tool name constants and disallowed-tool lists for agents |
