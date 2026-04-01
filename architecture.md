# Core Architecture & Orchestration

This document describes the engine that powers Claude Code locally: from user input to API call, stream parsing, tool execution, and back. All file references are relative to the repository root.

---

## 1. QueryEngine

**File:** `QueryEngine.ts`

The `QueryEngine` class owns the full lifecycle of a conversation. One instance is created per conversation; each call to `submitMessage()` represents a new turn within that conversation.

### Class Shape

```ts
export class QueryEngine {
  private config: QueryEngineConfig
  private mutableMessages: Message[]
  private abortController: AbortController
  private permissionDenials: SDKPermissionDenial[]
  private totalUsage: NonNullableUsage           // cumulative token usage across turns
  private hasHandledOrphanedPermission: boolean
  private readFileState: FileStateCache
  private discoveredSkillNames: Set<string>       // turn-scoped, cleared each submitMessage
  private loadedNestedMemoryPaths: Set<string>
}
```

`QueryEngineConfig` (line ~130) bundles everything the engine needs: `cwd`, `tools`, `commands`, `mcpClients`, `agents`, `canUseTool`, `getAppState`/`setAppState`, optional `customSystemPrompt`, `appendSystemPrompt`, `thinkingConfig`, `maxTurns`, `maxBudgetUsd`, `taskBudget`, `jsonSchema`, and more.

### Constructor (line ~200)

Initializes `mutableMessages` from `config.initialMessages ?? []`, creates the `abortController`, sets `totalUsage` to `EMPTY_USAGE`, and initializes empty `permissionDenials`.

### `submitMessage()` -- The Core Generator (line ~209)

```ts
async *submitMessage(
  prompt: string | ContentBlockParam[],
  options?: { uuid?: string; isMeta?: boolean },
): AsyncGenerator<SDKMessage, void, unknown>
```

This is an **async generator** that yields `SDKMessage` events (assistant text, tool results, system messages, compact boundaries, result summaries) to the caller (SDK or REPL). The caller consumes these via `for await ... of`.

Key phases inside `submitMessage`:

1. **System prompt construction** (lines ~284-325): Calls `fetchSystemPromptParts()` to get `defaultSystemPrompt`, `userContext`, and `systemContext`. Merges in coordinator context (`getCoordinatorUserContext`), memory mechanics prompt (when `CLAUDE_COWORK_MEMORY_PATH_OVERRIDE` is set), and `appendSystemPrompt`. Assembles with `asSystemPrompt([...])`.

2. **User input processing** (lines ~410-428): Calls `processUserInput()` to handle slash commands, produce messages, and determine whether an API query is needed (`shouldQuery`). Pushes resulting messages into `mutableMessages`.

3. **Transcript persistence** (lines ~450-463): Writes messages to `sessionStorage` via `recordTranscript()` before entering the API loop, so `/resume` works even if the process is killed mid-request.

4. **Query loop** (lines ~675+): Iterates over `query()` (from `query.ts`), processing each yielded message. Tracks `currentMessageUsage` via `accumulateUsage()` and `updateUsage()` (from `services/api/claude.ts`). Records assistant, user, and compact-boundary messages. Handles structured output tool calls, snip-boundary replays, and the final `result` yield.

5. **Result emission**: After the query loop terminates, yields a final `SDKResult` with `duration_ms`, `total_cost_usd`, `usage`, `modelUsage`, `permission_denials`, and `fast_mode_state`.

### State Tracking

| Field | Purpose |
|---|---|
| `mutableMessages` | Conversation history; mutated in-place as new messages arrive |
| `permissionDenials` | Tracks every tool use the `canUseTool` callback denied |
| `totalUsage` | Cumulative `NonNullableUsage` across all turns |
| `discoveredSkillNames` | Turn-scoped set of skill names discovered during tool execution |
| `readFileState` | `FileStateCache` for deduplicating file reads |
| `loadedNestedMemoryPaths` | Tracks which nested CLAUDE.md files have been loaded |

---

## 2. Message Loop Lifecycle

**Files:** `query.ts`, `services/api/claude.ts`, `services/tools/toolOrchestration.ts`

### `query()` (query.ts, line ~219)

```ts
export async function* query(params: QueryParams): AsyncGenerator<
  StreamEvent | RequestStartEvent | Message | TombstoneMessage | ToolUseSummaryMessage,
  Terminal
>
```

This is a thin wrapper that delegates to `queryLoop()` and fires command-lifecycle `completed` notifications for consumed commands.

### `queryLoop()` (query.ts, line ~241)

The heart of the agentic loop. Carries mutable `State` between iterations:

```ts
type State = {
  messages: Message[]
  toolUseContext: ToolUseContext
  autoCompactTracking: AutoCompactTrackingState | undefined
  maxOutputTokensRecoveryCount: number
  hasAttemptedReactiveCompact: boolean
  maxOutputTokensOverride: number | undefined
  pendingToolUseSummary: Promise<ToolUseSummaryMessage | null> | undefined
  stopHookActive: boolean | undefined
  turnCount: number
  transition: Continue | undefined        // why the previous iteration continued
}
```

Each iteration of the `while (true)` loop:

1. **Skill discovery prefetch** (line ~331): Fires `startSkillDiscoveryPrefetch()` in background.

2. **Message preparation**:
   - `getMessagesAfterCompactBoundary()` -- takes only messages after the last compaction.
   - `applyToolResultBudget()` -- enforces per-message budget on tool result sizes.
   - `snipCompactIfNeeded()` -- (feature-gated) trims old history via snip compaction.
   - `deps.microcompact()` -- removes stale tool results to save tokens.
   - `contextCollapse.applyCollapsesIfNeeded()` -- (feature-gated) projects collapsed context.
   - `deps.autocompact()` -- triggers full compaction when token count exceeds threshold. Yields `compactionResult` with `preCompactTokenCount`, `postCompactTokenCount`, and `summaryMessages`.

3. **System prompt finalization** (line ~449):
   ```ts
   const fullSystemPrompt = asSystemPrompt(
     appendSystemContext(systemPrompt, systemContext),
   )
   ```

4. **Model selection** (line ~572): `getRuntimeMainLoopModel()` resolves the model considering permission mode, plan mode, and token count.

5. **API call** (line ~659): Calls `deps.callModel()` (which is `queryModelWithStreaming` in production), passing `messages`, `systemPrompt`, `thinkingConfig`, `tools`, `signal`, and an `options` object with model, fast mode, tool choice, fallback model, effort, advisor, task budget, and query tracking.

6. **Stream consumption** (lines ~660-863): Iterates over the streaming response. For each `AssistantMessage`:
   - Backfills tool_use inputs via `tool.backfillObservableInput`.
   - Withholds recoverable errors (prompt-too-long, max-output-tokens) for potential recovery.
   - Pushes to `assistantMessages` array.
   - Extracts `tool_use` blocks and feeds them to `StreamingToolExecutor` for parallel execution.
   - Yields completed tool results from the streaming executor as they finish.

7. **Tool execution** (post-streaming): If `StreamingToolExecutor` is not used, falls back to `runTools()` (`services/tools/toolOrchestration.ts`, line ~19), which partitions tool calls into concurrency-safe batches (read-only tools run in parallel, up to `MAX_TOOL_USE_CONCURRENCY=10`) and sequential blocks.

8. **Stop hooks** (line ~999+): `executePostSamplingHooks()` runs after model response. `handleStopHooks()` decides if the loop should terminate or continue.

9. **Loop continuation**: If `needsFollowUp` is true (tool_use blocks were present), the loop continues with updated messages. Otherwise, terminal reason is returned.

10. **Error recovery**: The loop handles `FallbackTriggeredError` (switches to fallback model, clears assistant messages, retries), reactive compaction on prompt-too-long, and `max_output_tokens` recovery (up to `MAX_OUTPUT_TOKENS_RECOVERY_LIMIT = 3` retries).

### Streaming Tool Execution

`StreamingToolExecutor` (`services/tools/StreamingToolExecutor.ts`) begins executing tools as soon as their `tool_use` blocks arrive from the stream, before the full response is complete. Tools are added via `addTool()` and completed results are polled via `getCompletedResults()` during the stream loop.

---

## 3. API Services

**Files:** `services/api/claude.ts`, `services/api/client.ts`, `services/api/withRetry.ts`, `services/api/errors.ts`

### Client Construction (client.ts)

`getAnthropicClient()` creates an `Anthropic` SDK instance configured for the active provider:
- **First-party (1P)**: Direct Anthropic API with `ANTHROPIC_API_KEY` or OAuth tokens.
- **AWS Bedrock**: AWS credentials, region-aware routing via `AWS_REGION` / model-specific region vars.
- **Google Vertex**: GCP credentials, `ANTHROPIC_VERTEX_PROJECT_ID`, region-aware routing.
- **Azure Foundry**: `ANTHROPIC_FOUNDRY_RESOURCE` or `ANTHROPIC_FOUNDRY_BASE_URL`.

### `queryModelWithStreaming()` (claude.ts, line ~752)

```ts
export async function* queryModelWithStreaming({
  messages, systemPrompt, thinkingConfig, tools, signal, options,
}: { ... }): AsyncGenerator<StreamEvent | AssistantMessage | SystemAPIErrorMessage, void>
```

Wraps `queryModel()` inside `withStreamingVCR()` (for recording/playback in tests). The inner `queryModel()`:

- Normalizes messages for the API (`normalizeMessagesForAPI`).
- Applies prompt caching (`getCacheControl()` with ephemeral TTL, optional 1-hour for eligible users).
- Configures thinking (`thinkingConfig: adaptive | enabled | disabled`), effort, task budget, structured outputs.
- Sets up beta headers: `FAST_MODE_BETA_HEADER`, `CONTEXT_1M_BETA_HEADER`, `EFFORT_BETA_HEADER`, `TASK_BUDGETS_BETA_HEADER`, etc.
- Calls `anthropic.beta.messages.stream()` and iterates over `BetaRawMessageStreamEvent`s.
- Parses delta events (`content_block_start`, `content_block_delta`, `content_block_stop`, `message_start`, `message_delta`, `message_stop`).
- Accumulates usage via `updateUsage()`.
- Calls `addToTotalSessionCost()` on `message_stop`.
- Falls back to non-streaming (`executeNonStreamingRequest`) on certain stream failures.

### Retry Logic (withRetry.ts)

```ts
export async function* withRetry<T>(
  getClient: () => Promise<Anthropic>,
  operation: (client: Anthropic, attempt: number, context: RetryContext) => Promise<T>,
  options: RetryOptions,
): AsyncGenerator<SystemAPIErrorMessage, T>
```

- **Max retries**: `DEFAULT_MAX_RETRIES = 10`.
- **Base delay**: `BASE_DELAY_MS = 500`, with exponential backoff.
- **529 (overloaded)**: Up to `MAX_529_RETRIES = 3` for foreground query sources (defined in `FOREGROUND_529_RETRY_SOURCES`). Background sources (summaries, classifiers) bail immediately to avoid amplification.
- **429 (rate limit)**: Respects `Retry-After` header. For fast mode, short delays retry with fast mode active; long delays trigger cooldown and fall back to standard speed.
- **401/403**: Refreshes OAuth tokens via `handleOAuth401Error()`, then retries with a fresh client.
- **Bedrock/Vertex auth errors**: Refreshes provider-specific credentials.
- **ECONNRESET/EPIPE**: Disables keep-alive and reconnects.
- **Persistent retry** (`CLAUDE_CODE_UNATTENDED_RETRY`): For unattended sessions, retries 429/529 indefinitely with `PERSISTENT_MAX_BACKOFF_MS = 5min`, heartbeat yields every 30s.
- **Model fallback**: Throws `FallbackTriggeredError` when consecutive 529s exceed the limit, caught by `queryLoop()` to switch models.

### Error Handling (errors.ts)

`categorizeRetryableAPIError()` classifies errors for the query loop. `getAssistantMessageFromError()` converts API errors into synthetic `AssistantMessage`s. `parsePromptTooLongTokenCounts()` extracts actual/limit from "prompt is too long: 137500 tokens > 135000 maximum" messages.

---

## 4. Dynamic System Prompts

**Files:** `utils/queryContext.ts`, `context.ts`, `constants/prompts.ts`

System prompts are constructed fresh each turn from multiple layers:

### `fetchSystemPromptParts()` (utils/queryContext.ts, line ~44)

```ts
export async function fetchSystemPromptParts({
  tools, mainLoopModel, additionalWorkingDirectories, mcpClients, customSystemPrompt,
}): Promise<{
  defaultSystemPrompt: string[]
  userContext: { [k: string]: string }
  systemContext: { [k: string]: string }
}>
```

Runs three calls in parallel:
1. `getSystemPrompt(tools, mainLoopModel, additionalWorkingDirectories, mcpClients)` -- from `constants/prompts.ts`. Builds the full base system prompt including tool descriptions, model-specific instructions, and MCP server tool schemas. **Skipped** when `customSystemPrompt` is provided.
2. `getUserContext()` -- from `context.ts`.
3. `getSystemContext()` -- from `context.ts`. **Skipped** when `customSystemPrompt` is provided.

### `getUserContext()` (context.ts, line ~155)

Memoized for the session. Returns:
- **`claudeMd`**: Combined contents of all discovered CLAUDE.md files. Discovery walks from cwd upward and into `~/.claude/`. Controlled by `CLAUDE_CODE_DISABLE_CLAUDE_MDS` env var. In bare mode, only explicit `--add-dir` paths are scanned.
- **`currentDate`**: `"Today's date is YYYY-MM-DD."`

### `getSystemContext()` (context.ts, line ~116)

Memoized for the session. Returns:
- **`gitStatus`**: Snapshot of branch, main branch, git user, `git status --short` (truncated at 2000 chars), and last 5 commits. Skipped for CCR (remote) sessions or when git instructions are disabled.
- **`cacheBreaker`**: (ant-only) Injected string for cache invalidation debugging.

### Assembly in QueryEngine (QueryEngine.ts, lines ~302-325)

```ts
const userContext = {
  ...baseUserContext,
  ...getCoordinatorUserContext(mcpClients, scratchpadDir),
}

const systemPrompt = asSystemPrompt([
  ...(customPrompt !== undefined ? [customPrompt] : defaultSystemPrompt),
  ...(memoryMechanicsPrompt ? [memoryMechanicsPrompt] : []),
  ...(appendSystemPrompt ? [appendSystemPrompt] : []),
])
```

The `userContext` dict is prepended to messages via `prependUserContext()` in query.ts (line ~660). The `systemContext` dict is appended to the system prompt via `appendSystemContext()` (line ~449).

### Memory Integration

- `loadMemoryPrompt()` (from `memdir/memdir.ts`) injects memory mechanics when `CLAUDE_COWORK_MEMORY_PATH_OVERRIDE` is set.
- CLAUDE.md files are discovered via `getMemoryFiles()` and combined by `getClaudeMds()`. The result is cached in bootstrap state via `setCachedClaudeMdContent()` for downstream consumers (auto-mode classifier).

---

## 5. Cost Tracking

**Files:** `cost-tracker.ts`, `utils/modelCost.ts`, `services/api/claude.ts` (lines ~2924-3020), `bootstrap/state.ts`

### Token Counting

Usage is tracked at two levels:

1. **Per-message**: `updateUsage()` (claude.ts:2924) merges `BetaMessageDeltaUsage` into `NonNullableUsage`, guarding against zero-overwrites from `message_delta` events. Fields: `input_tokens`, `output_tokens`, `cache_creation_input_tokens`, `cache_read_input_tokens`, `server_tool_use.web_search_requests`, `service_tier`, `speed`.

2. **Per-turn cumulative**: `accumulateUsage()` (claude.ts:2993) sums two `NonNullableUsage` objects. Used by `QueryEngine` to maintain `totalUsage` across the turn.

### Cost Calculation

`calculateUSDCost()` (`utils/modelCost.ts`) maps model names to pricing tiers:

| Tier | Input | Output | Cache Write | Cache Read | Web Search |
|---|---|---|---|---|---|
| Sonnet ($3/$15) | $3/MTok | $15/MTok | $3.75/MTok | $0.30/MTok | $0.01/req |
| Opus 4/4.1 ($15/$75) | $15/MTok | $75/MTok | $18.75/MTok | $1.50/MTok | $0.01/req |
| Opus 4.5 ($5/$25) | $5/MTok | $25/MTok | $6.25/MTok | $0.50/MTok | $0.01/req |

Model configs are defined in `utils/model/configs.ts` and mapped via `getCanonicalName()`.

### Session Cost Tracking

`addToTotalSessionCost()` (cost-tracker.ts:278):
```ts
export function addToTotalSessionCost(cost: number, usage: Usage, model: string): number
```
- Calls `addToTotalModelUsage()` to accumulate per-model usage (input, output, cache read, cache creation, web search, cost).
- Calls `addToTotalCostState()` to update global counters in `bootstrap/state.ts`.
- Reports to OpenTelemetry counters (`getCostCounter()`, `getTokenCounter()`).
- Processes advisor tool usage separately (`getAdvisorUsage()`).

### Session Persistence

`saveCurrentSessionCosts()` (cost-tracker.ts:143) writes to project config:
- `lastCost`, `lastAPIDuration`, `lastToolDuration`, `lastDuration`
- `lastLinesAdded`, `lastLinesRemoved`
- `lastTotalInputTokens`, `lastTotalOutputTokens`
- `lastModelUsage` (per-model breakdown)
- `lastSessionId` (for restore validation)

`restoreCostStateForSession()` (cost-tracker.ts:130) restores on `/resume` if session IDs match.

`formatTotalCost()` (cost-tracker.ts:228) renders the session summary with per-model usage breakdown.

---

## 6. State Management

**Files:** `state/AppStateStore.ts`, `state/AppState.tsx`, `state/store.ts`

### `AppState` (AppStateStore.ts, line ~89)

`AppState` is a `DeepImmutable` record containing the full UI-reactive state:

```ts
export type AppState = DeepImmutable<{
  settings: SettingsJson
  verbose: boolean
  mainLoopModel: ModelSetting
  mainLoopModelForSession: ModelSetting
  toolPermissionContext: ToolPermissionContext
  kairosEnabled: boolean                     // assistant daemon mode
  // MCP state
  mcp: { clients, tools, commands, resources, pluginReconnectKey }
  // Bridge/remote state
  replBridgeEnabled: boolean
  replBridgeConnected: boolean
  // Task state
  tasks: { [taskId: string]: TaskState }
  agentNameRegistry: Map<string, AgentId>
  // UI state
  expandedView: 'none' | 'tasks' | 'teammates'
  footerSelection: FooterItem | null
  // Speculation
  speculation: SpeculationState
  // File history, attribution, denial tracking, ...
}>
```

### Store (state/store.ts)

`createStore()` returns a `Store<AppState>` with `getState()`, `setState()`, and `subscribe()`. The React side exposes `AppStoreContext` and the `useAppState(selector)` hook (AppState.tsx:142), which uses `useSyncExternalStore` for selective re-renders via `Object.is` comparison.

### Fast Mode

**File:** `utils/fastMode.ts`

Fast mode routes requests to the Opus 4.6 model at higher throughput. Key state:

- **Availability**: `isFastModeEnabled()` checks `CLAUDE_CODE_DISABLE_FAST_MODE` env var. `isFastModeAvailable()` additionally checks org status, API provider (1P only), SDK restrictions, and Statsig gates.
- **Org status**: `FastModeOrgStatus` = `pending | enabled | disabled(reason)`. Fetched via `prefetchFastModeStatus()` which calls `/api/claude_code_penguin_mode`. Cached in memory and persisted to `penguinModeOrgEnabled` in global config.
- **Runtime state**: `FastModeRuntimeState` = `active | cooldown(resetAt, reason)`. Cooldown is triggered by 429/529 errors with long retry-after via `triggerFastModeCooldown()`.
- **State function**: `getFastModeState(model, fastModeUserEnabled)` returns `'off' | 'cooldown' | 'on'`.
- **Model**: `getFastModeModel()` returns `'opus'` (or `'opus[1m]'` when 1M merge is enabled). Display name: `FAST_MODE_MODEL_DISPLAY = 'Opus 4.6'`.
- **Model support**: `isFastModeSupportedByModel()` checks if the model string contains `'opus-4-6'`.

### Thinking Mode

**File:** `utils/thinking.ts`

```ts
export type ThinkingConfig =
  | { type: 'adaptive' }           // model decides when to think
  | { type: 'enabled'; budgetTokens: number }  // always think with budget
  | { type: 'disabled' }           // no thinking
```

- `shouldEnableThinkingByDefault()` checks build flags, model capability, GrowthBook gates, and settings.
- `modelSupportsThinking(model)` checks 3P overrides, ant model aliases, and canonical model names.
- `isUltrathinkEnabled()` gates the "ultrathink" feature (extended thinking with higher budgets).
- `hasUltrathinkKeyword(text)` detects `/\bultrathink\b/i` in user input for trigger detection.

---

## 7. Startup & Initialization

**File:** `setup.ts`

```ts
export async function setup(
  cwd: string,
  permissionMode: PermissionMode,
  allowDangerouslySkipPermissions: boolean,
  worktreeEnabled: boolean,
  worktreeName: string | undefined,
  tmuxEnabled: boolean,
  customSessionId?: string | null,
  worktreePRNumber?: number,
  messagingSocketPath?: string,
): Promise<void>
```

Bootstrap sequence:

1. **Node.js version check**: Requires >= 18, exits with error otherwise.
2. **Session ID**: If `customSessionId` is provided, calls `switchSession()`.
3. **UDS messaging server**: Starts Unix Domain Socket messaging for inter-process communication (bare mode skipped unless explicit socket path).
4. **Teammate snapshot**: Captures agent swarm teammate state (bare mode skipped).
5. **Terminal backup restoration**: Restores iTerm2 and Terminal.app settings if a previous session was interrupted.
6. **`setCwd(cwd)`**: Sets the working directory globally.
7. **Hooks config snapshot**: `captureHooksConfigSnapshot()` records hook configuration to detect hidden modifications.
8. **File changed watcher**: `initializeFileChangedWatcher(cwd)` sets up filesystem monitoring for hooks.
9. **Worktree creation**: If `--worktree` flag, creates a git worktree via `createWorktreeForSession()`. Resolves canonical git root, handles tmux session creation.
10. **Release notes check**: `checkForReleaseNotes()`.
11. **Session memory init**: `initSessionMemory()`.
12. **API key prefetch**: `prefetchApiKeyFromApiKeyHelperIfSafe()`.

---

## 8. Key Source Files Reference

| File | Purpose |
|---|---|
| `QueryEngine.ts` | Conversation lifecycle, `submitMessage()` generator, state management |
| `query.ts` | Inner agentic loop (`queryLoop`), message preparation, compaction orchestration, tool execution flow |
| `services/api/claude.ts` | `queryModelWithStreaming()`, API request assembly, streaming parser, usage tracking (`updateUsage`, `accumulateUsage`) |
| `services/api/client.ts` | `getAnthropicClient()` -- SDK client construction for 1P/Bedrock/Vertex/Foundry |
| `services/api/withRetry.ts` | `withRetry()` generator -- exponential backoff, 429/529 handling, fast mode fallback, auth refresh |
| `services/api/errors.ts` | Error classification, prompt-too-long parsing, error-to-message conversion |
| `services/tools/toolOrchestration.ts` | `runTools()` -- partitions tool calls into concurrent/sequential batches |
| `services/tools/StreamingToolExecutor.ts` | Executes tools as soon as `tool_use` blocks arrive from stream |
| `cost-tracker.ts` | Session cost tracking, persistence, formatting |
| `utils/modelCost.ts` | `calculateUSDCost()`, per-model pricing tiers |
| `utils/queryContext.ts` | `fetchSystemPromptParts()` -- assembles system prompt, user context, system context |
| `context.ts` | `getUserContext()` (CLAUDE.md), `getSystemContext()` (git status) |
| `state/AppStateStore.ts` | `AppState` type definition, `getDefaultAppState()` |
| `state/AppState.tsx` | React provider, `useAppState()` selector hook |
| `utils/fastMode.ts` | Fast mode availability, org status prefetch, cooldown management |
| `utils/thinking.ts` | `ThinkingConfig` type, model support detection, ultrathink |
| `setup.ts` | Bootstrap sequence: env checks, worktree, hooks, session init |
| `utils/sessionStorage.ts` | `recordTranscript()`, `flushSessionStorage()` -- conversation persistence |
| `memdir/memdir.ts` | `loadMemoryPrompt()` -- memory mechanics injection |
| `utils/processUserInput/processUserInput.ts` | Slash command processing, attachment handling, model override extraction |
| `services/compact/autoCompact.ts` | Auto-compaction threshold calculation, token warning states |
| `services/compact/compact.ts` | `buildPostCompactMessages()` -- assembles messages after compaction |
| `query/tokenBudget.ts` | Token budget tracking for auto-continue feature |
| `query/stopHooks.ts` | Stop hook evaluation (decides if loop should continue) |
| `query/config.ts` | `buildQueryConfig()` -- snapshots env/session state once per turn |
