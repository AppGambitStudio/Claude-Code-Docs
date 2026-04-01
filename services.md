# Services and Integrations

The `services/` directory contains the core subsystems that support the main REPL loop. Each service is designed as a self-contained module with a clear public API, communicating through well-defined boundaries rather than tight coupling. Services range from infrastructure concerns (analytics, cost tracking, rate limits) to feature-level capabilities (MCP, voice, LSP, compaction).

This document covers each service in depth, including key functions, data flow, and integration points.

---

## 1. MCP Integration

**Directory:** `services/mcp/`

The Model Context Protocol subsystem connects Claude Code to external tool servers, exposing their capabilities as native tools within the conversation loop. It is one of the largest services, spanning client lifecycle management, configuration resolution, authentication, and tool execution.

### Client lifecycle

`services/mcp/client.ts` is the primary module. It implements the full MCP client using `@modelcontextprotocol/sdk`:

- **Transport support** -- Stdio (`StdioClientTransport`), SSE (`SSEClientTransport`), Streamable HTTP (`StreamableHTTPClientTransport`), WebSocket (`WebSocketTransport`), and an in-process SDK control transport (`SdkControlClientTransport`).
- **`ensureConnectedClient()`** -- Lazily connects to an MCP server, caching the client instance per server name. Handles reconnection on auth expiry.
- **`getMcpToolsCommandsAndResources()`** -- After connection, fetches all tools, commands, and resources from a server. Tools are wrapped as `MCPTool` instances, resources as `ServerResource` objects. Commands from MCP prompts are also extracted.
- **`fetchToolsForClient()` / `fetchResourcesForClient()` / `fetchCommandsForClient()`** -- Individual fetch functions for each capability type.
- **`callMCPToolWithUrlElicitationRetry()`** -- Executes an MCP tool call with automatic retry when the server requests OAuth elicitation.
- **`processMCPResult()` / `transformMCPResult()`** -- Processes raw MCP tool call results into Claude-compatible content blocks, handling binary blobs, image resizing, large output truncation, and Unicode sanitization.
- **`reconnectMcpServerImpl()`** -- Tears down and re-establishes a server connection, returning fresh tools/commands/resources.
- **`clearServerCache()`** -- Removes cached client state for a server.
- **`callIdeRpc()`** -- Sends RPC calls to IDE-connected MCP servers (used for diagnostics, file sync, etc.).

### Configuration

`services/mcp/config.ts` resolves MCP server configuration from multiple scopes:

- **Scopes:** `local`, `user`, `project`, `dynamic`, `enterprise`, `claudeai`, `managed` (defined in `ConfigScope` type in `types.ts`).
- **`getAllMcpConfigs()`** -- Merges server configs from all scopes with precedence rules. Enterprise configs come from `managed-mcp.json`; project configs from `.mcp.json`; user configs from global settings.
- **`getMcpConfigsByScope()`** -- Returns servers filtered to a single scope.
- **`getEnterpriseMcpFilePath()`** -- Returns path to the enterprise-managed MCP config file.
- **`isMcpServerDisabled()` / `setMcpServerEnabled()`** -- Per-server enable/disable toggle.
- **`writeMcpjsonFile()`** -- Atomic write to `.mcp.json` with permission preservation and temp-file rename.
- **Server config types:** `McpStdioServerConfig`, `McpSSEServerConfig`, `McpHTTPServerConfig`, `McpWebSocketServerConfig` (all defined in `types.ts` with Zod schemas).

### Transport types

`services/mcp/types.ts` defines transport variants: `stdio`, `sse`, `sse-ide`, `http`, `ws`, `sdk`. Each has its own Zod schema for validation. Server configs include optional OAuth configuration (`McpOAuthConfigSchema`) and Cross-App Access (XAA) flags.

### Connection management (React layer)

`services/mcp/MCPConnectionManager.tsx` provides a React context wrapping `useManageMCPConnections()`. Exposes:

- **`useMcpReconnect()`** -- Hook for reconnecting a specific server by name.
- **`useMcpToggleEnabled()`** -- Hook for toggling a server on/off.

`services/mcp/useManageMCPConnections.ts` contains the connection management logic, including:

- Listening for `ToolListChangedNotification`, `ResourceListChangedNotification`, and `PromptListChangedNotification` from the MCP SDK to auto-refresh capabilities.
- Deduplication of Claude.ai MCP servers (`dedupClaudeAiMcpServers()`).
- Enterprise MCP config detection (`doesEnterpriseMcpConfigExist()`).
- Policy-based server filtering (`filterMcpServersByPolicy()`).

### Auth approvals

`services/mcpServerApproval.tsx` renders Ink-based approval dialogs when project-scoped MCP servers need user consent:

- **`handleMcpjsonServerApprovals()`** -- Checks for pending project servers and renders either `MCPServerApprovalDialog` (single server) or `MCPServerMultiselectDialog` (multiple servers).

`services/mcp/auth.ts` implements `ClaudeAuthProvider` for OAuth-based MCP server authentication, including step-up detection and token refresh.

### Other MCP modules

| File | Purpose |
|------|---------|
| `channelPermissions.ts` | Permission checks for MCP channels |
| `channelAllowlist.ts` | Allowlist management for MCP server channels |
| `channelNotification.ts` | Notification routing for channel events |
| `elicitationHandler.ts` | Handles MCP elicitation requests (OAuth prompts to users) |
| `envExpansion.ts` | Environment variable expansion in server configs |
| `headersHelper.ts` | Custom header resolution for HTTP-based MCP servers |
| `mcpStringUtils.ts` | `buildMcpToolName()` and name formatting utilities |
| `normalization.ts` | `normalizeNameForMCP()` -- sanitizes server/tool names |
| `officialRegistry.ts` | Integration with the official MCP server registry |
| `oauthPort.ts` | OAuth callback port management |
| `InProcessTransport.ts` | Transport for in-process MCP servers |
| `SdkControlTransport.ts` | Transport for SDK-controlled MCP connections |
| `vscodeSdkMcp.ts` | VS Code SDK MCP integration |
| `xaa.ts` / `xaaIdpLogin.ts` | Cross-App Access (XAA) identity provider login |
| `claudeai.ts` | Claude.ai-specific MCP config fetching |
| `utils.ts` | `getProjectMcpServerStatus()`, `getLoggingSafeMcpBaseUrl()` |

---

## 2. Analytics and Diagnostics

**Directory:** `services/analytics/`

The analytics subsystem provides sanitized, privacy-aware event logging with zero-dependency design to avoid import cycles.

### Event logging API

`services/analytics/index.ts` is the public entry point. It exposes:

- **`logEvent(eventName, metadata)`** -- Synchronous event logging. Events are queued in memory until a sink is attached.
- **`logEventAsync(eventName, metadata)`** -- Async variant for events that need delivery confirmation.
- **`attachAnalyticsSink(sink)`** -- Called during app startup to attach the routing backend. Idempotent. Queued events are drained via `queueMicrotask` to avoid blocking startup.
- **`stripProtoFields(metadata)`** -- Removes `_PROTO_*` keys from metadata before routing to general-access storage. Proto-prefixed keys are only sent to the first-party exporter where they map to PII-tagged proto columns.

**Safety types:**

- `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` -- A `never`-typed brand that forces developers to explicitly cast string values, confirming they do not contain code or file paths.
- `AnalyticsMetadata_I_VERIFIED_THIS_IS_PII_TAGGED` -- Brand for values destined for PII-tagged proto columns with restricted access.

### Sink implementation

`services/analytics/sink.ts` routes events to two backends:

- **Datadog** (`datadog.ts`) -- Gated by a feature flag (`tengu_log_datadog_events`). Uses `trackDatadogEvent()`.
- **First-party event logger** (`firstPartyEventLogger.ts`) -- Logs to Anthropic's internal event pipeline. Applies sampling via `shouldSampleEvent()`.
- **Kill switch** (`sinkKillswitch.ts`) -- Can disable individual sinks at runtime.

**`initializeAnalyticsSink()`** wires up the sink during app startup.

### Supporting modules

| File | Purpose |
|------|---------|
| `config.ts` | Analytics configuration |
| `growthbook.ts` | Feature flag integration via GrowthBook. `getFeatureValue_CACHED_MAY_BE_STALE()` is used throughout the codebase for gating features |
| `metadata.ts` | Event metadata enrichment |

### Diagnostic tracking

`services/diagnosticTracking.ts` provides `DiagnosticTrackingService`, a singleton that tracks IDE diagnostics (errors/warnings) across files:

- **`initialize(mcpClient)`** -- Connects to an IDE MCP client to fetch diagnostics.
- **`shutdown()` / `reset()`** -- Lifecycle management for diagnostic state.
- Maintains a baseline of diagnostics and tracks changes introduced by Claude's edits.
- Normalizes file URIs and caps diagnostic summaries at 4000 characters.

---

## 3. Cost Tracking

**Files:** `cost-tracker.ts`, `costHook.ts`

Cost tracking aggregates token usage and USD costs across the entire session, persisting state between sessions and displaying summaries on exit.

### Core functions in `cost-tracker.ts`

- **`getStoredSessionCosts(sessionId)`** -- Reads persisted cost state from project config. Only returns data if the session ID matches the last saved session.
- **`restoreCostStateForSession(sessionId)`** -- Restores accumulated cost state when resuming a session. Returns `true` if restoration succeeded.
- **`saveCurrentSessionCosts(fpsMetrics?)`** -- Persists current cost counters to project config. Saves: total cost, API duration, tool duration, lines added/removed, per-model token breakdowns, web search request counts, and FPS metrics.
- **`formatTotalCost()`** -- Renders a chalk-formatted summary including cost, API/wall duration, code change stats, and per-model usage breakdown.
- **`formatCost(cost, maxDecimalPlaces)`** -- Formats a USD value, switching between 2 and 4 decimal places based on magnitude.

### State management

Cost state lives in `bootstrap/state.ts` with accessor functions re-exported from `cost-tracker.ts`:

| Function | Description |
|----------|-------------|
| `getTotalCostUSD()` | Accumulated USD cost for the session |
| `getTotalInputTokens()` / `getTotalOutputTokens()` | Total token counts |
| `getTotalCacheReadInputTokens()` / `getTotalCacheCreationInputTokens()` | Prompt cache statistics |
| `getTotalAPIDuration()` / `getTotalAPIDurationWithoutRetries()` | API call timing |
| `getTotalLinesAdded()` / `getTotalLinesRemoved()` | Code change tracking |
| `getModelUsage()` / `getUsageForModel(model)` | Per-model `ModelUsage` records (input/output/cache tokens, cost, web searches) |
| `hasUnknownModelCost()` | Whether any model lacked a pricing entry |

Actual USD calculation is performed by `calculateUSDCost()` in `utils/modelCost.ts`, which maps model names to per-token prices for input, output, cache read, and cache creation.

### StoredCostState

The `StoredCostState` type persisted to project config includes: `totalCostUSD`, `totalAPIDuration`, `totalAPIDurationWithoutRetries`, `totalToolDuration`, `totalLinesAdded`, `totalLinesRemoved`, `lastDuration`, and `modelUsage` (keyed by model name).

### The cost hook

`costHook.ts` contains `useCostSummary()`, a React hook that:

1. Registers a `process.on('exit')` handler.
2. On exit, prints the formatted cost summary (only if the user has console billing access via `hasConsoleBillingAccess()`).
3. Calls `saveCurrentSessionCosts()` to persist state for session resumption.

---

## 4. Voice I/O

**Files:** `services/voice.ts`, `services/voiceStreamSTT.ts`, `services/voiceKeyterms.ts`

Voice input provides push-to-talk speech-to-text for the CLI, using native audio capture with fallback to external tools.

### Audio recording (`voice.ts`)

Recording uses a layered capture strategy:

1. **Native audio capture** (`audio-capture-napi`) -- Preferred on macOS, Linux, and Windows. Links against CoreAudio/AudioUnit frameworks. Lazy-loaded on first voice keypress via `loadAudioNapi()` to avoid blocking startup (dlopen takes 1-8s).
2. **SoX `rec`** -- Fallback on Linux. Detected via `hasCommand('rec')`.
3. **ALSA `arecord`** -- Second fallback on Linux. Validated with `probeArecord()` which spawns arecord for 150ms to test device availability (handles WSL/headless scenarios).

**Constants:** 16kHz sample rate, mono channel, 2.0s silence detection duration, 3% silence threshold.

`checkRecordingAvailability()` probes all capture backends in priority order and returns whether recording is possible on the current platform.

### Speech-to-text streaming (`voiceStreamSTT.ts`)

Connects to Anthropic's `voice_stream` WebSocket endpoint at `/api/ws/speech_to_text/voice_stream`:

- **Wire protocol:** JSON control messages (`KeepAlive`, `CloseStream`) and binary audio frames. Server responds with `TranscriptText` (partial/final text) and `TranscriptEndpoint` (end-of-utterance) messages.
- **`VoiceStreamConnection`** -- The connection interface with `send(audioChunk)`, `finalize()`, `close()`, and `isConnected()`.
- **`VoiceStreamCallbacks`** -- Callback interface: `onTranscript(text, isFinal)`, `onError(error)`, `onClose()`, `onReady(connection)`.
- **`isVoiceStreamAvailable()`** -- Checks if voice streaming is available (requires Anthropic OAuth authentication).
- **`FinalizeSource`** -- Indicates how transcription ended: `post_closestream_endpoint`, `no_data_timeout` (1500ms, silent-drop detection), `safety_timeout` (5000ms), `ws_close`, or `ws_already_closed`.
- **Keepalive:** 8-second interval to maintain the WebSocket connection.

### Voice keyterms (`voiceKeyterms.ts`)

Improves STT accuracy by providing domain-specific vocabulary hints (Deepgram keywords):

- **`getVoiceKeyterms(recentFiles?)`** -- Builds a list of up to `MAX_KEYTERMS` (50) terms from:
  - **Global keyterms:** Hardcoded coding terms that Deepgram consistently mangles (MCP, symlink, grep, regex, localhost, TypeScript, JSON, OAuth, webhook, gRPC, dotfiles, subagent, worktree).
  - **Project root basename** -- e.g., "claude-code" as a phrase.
  - **Git branch words** -- `splitIdentifier()` breaks `feat/voice-keyterms` into individual words.
  - **Recent file names** -- Extracts words from file stems.
- **`splitIdentifier(name)`** -- Splits camelCase, PascalCase, kebab-case, snake_case, and path segments into words. Filters to 3-20 character length.

---

## 5. Rate Limiting and Policy

**Files:** `services/rateLimitMessages.ts`, `services/rateLimitMocking.ts`, `services/mockRateLimits.ts`, `services/claudeAiLimits.ts`

### Rate limit messages

`services/rateLimitMessages.ts` is the single source of truth for rate limit user messages:

- **`getRateLimitMessage(limits, model)`** -- Returns the appropriate warning/error message based on `ClaudeAILimits` state (overage status, weekly limits, session limits).
- **`isRateLimitErrorMessage(text)`** -- Detects rate limit error messages by prefix matching against `RATE_LIMIT_ERROR_PREFIXES`.
- Handles overage scenarios, fallback model availability, and different subscription tiers.

### Mock rate limits (internal testing)

`services/mockRateLimits.ts` provides test scenarios for rate limit UI without hitting real API limits:

- **`MockScenario`** type covers scenarios: `normal`, `session-limit-reached`, `approaching-weekly-limit`, `weekly-limit-reached`, `overage-active`, `overage-warning`, `overage-exhausted`, `out-of-credits`, `opus-limit`, `sonnet-limit`, `fast-mode-limit`, and more.
- Generates mock `anthropic-ratelimit-unified-*` headers for each scenario.

`services/rateLimitMocking.ts` is the facade layer:

- **`processRateLimitHeaders(headers)`** -- Applies mock headers if the `/mock-limits` command is active, otherwise passes through real headers.
- **`shouldProcessRateLimits(isSubscriber)`** -- Returns true for subscribers or when mock limits are active.
- **`checkMockRateLimitError(currentModel, isFastModeActive)`** -- Generates synthetic 429 errors for mock scenarios.

### Policy limits

**Directory:** `services/policyLimits/`

Fetches organization-level policy restrictions from the API:

- **`initializePolicyLimitsLoadingPromise()`** -- Creates a loading promise early in startup for other systems to await. Includes a 30-second timeout to prevent deadlocks.
- **`loadPolicyLimits()`** -- Fetches restrictions from the API with ETag caching. Eligible users: all console (API key) users; Team and Enterprise OAuth users.
- **Polling:** Background polling every hour (`POLLING_INTERVAL_MS = 3,600,000ms`), with retry logic up to 5 attempts.
- **Fail-open:** If the API call fails, the CLI continues without restrictions.
- **Cache:** Session-level in-memory cache plus disk cache at `policy-limits.json` in the config home directory.

`services/policyLimits/types.ts` defines the response schema:

```typescript
type PolicyLimitsResponse = {
  restrictions: Record<string, { allowed: boolean }>
}
```

Only blocked policies are included in the response. Absent keys are treated as allowed.

---

## 6. Compaction Service

**Directory:** `services/compact/`

Compaction manages conversation context length by summarizing older messages when the token count approaches the model's context window limit.

### Auto-compaction trigger (`autoCompact.ts`)

- **`getEffectiveContextWindowSize(model)`** -- Returns context window minus reserved tokens for summary output (capped at 20,000 tokens based on p99.99 data).
- **`getAutoCompactThreshold(model)`** -- Effective context window minus `AUTOCOMPACT_BUFFER_TOKENS` (13,000). Overridable via `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` env var.
- **`calculateTokenWarningState(tokenUsage, model)`** -- Returns `percentLeft`, `isAboveWarningThreshold`, `isAboveErrorThreshold`, and `isAboveAutoCompactThreshold`.
- **Circuit breaker:** `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3` -- stops retrying after 3 consecutive failures to prevent wasting API calls.
- **Buffer constants:** `WARNING_THRESHOLD_BUFFER_TOKENS = 20,000`, `ERROR_THRESHOLD_BUFFER_TOKENS = 20,000`, `MANUAL_COMPACT_BUFFER_TOKENS = 3,000`.

### Core compaction (`compact.ts`)

- **`compactConversation()`** -- The main compaction function. Uses a forked subagent (`runForkedAgent()`) to summarize conversation history. Produces a `CompactionResult` containing the summary and boundary markers.
- **`partialCompactConversation()`** -- Partial compaction for incremental summarization (keeps recent messages intact).
- **`buildPostCompactMessages(result)`** -- Constructs the post-compaction message array with boundary markers.
- **`stripImagesFromMessages()`** -- Removes image content before compaction to save tokens.
- **`truncateHeadForPTLRetry()`** -- Truncates early messages when hitting prompt-too-long errors.
- **`createCompactCanUseTool()`** -- Returns a restricted `CanUseToolFn` for the compaction agent.

### Message grouping (`grouping.ts`)

- **`groupMessagesByApiRound(messages)`** -- Groups messages at API-round boundaries (one group per round-trip). Uses assistant message IDs as boundary markers. This enables reactive compaction on single-prompt agentic sessions where the entire workload is one human turn.

### Micro-compaction (`microCompact.ts`)

Time-based micro-compaction that clears old tool results without a full summarization pass:

- **`COMPACTABLE_TOOLS`** -- Only compacts results from: FileRead, Shell tools, Grep, Glob, WebSearch, WebFetch, FileEdit, FileWrite.
- **Time-based MC** clears tool result content with `[Old tool result content cleared]` markers.
- **Cached MC** (feature-gated) -- Uses a separate `cachedMicrocompact.js` module for cache-aware clearing.

### Supporting modules

| File | Purpose |
|------|---------|
| `prompt.ts` | Compaction prompt templates. Includes `NO_TOOLS_PREAMBLE` to prevent the summarizer from calling tools, plus `DETAILED_ANALYSIS_INSTRUCTION_BASE` / `DETAILED_ANALYSIS_INSTRUCTION_PARTIAL` for thorough summary generation |
| `postCompactCleanup.ts` | `runPostCompactCleanup()` -- cleanup after compaction |
| `sessionMemoryCompact.ts` | `trySessionMemoryCompaction()` -- compacts session memory alongside conversation |
| `compactWarningState.ts` | Manages warning suppression state (pure, React-free) |
| `compactWarningHook.ts` | `useCompactWarningSuppression()` -- React hook for warning state |
| `timeBasedMCConfig.ts` | `getTimeBasedMCConfig()` -- configuration for time-based micro-compaction |
| `apiMicrocompact.ts` | API-level micro-compaction implementation |

---

## 7. Magic Docs

**Directory:** `services/MagicDocs/`

Magic Docs automatically maintains markdown documentation files that are marked with a special header. When Claude reads a file containing `# MAGIC DOC: [title]`, the system registers it and periodically updates it in the background using a forked subagent.

### How it works

1. **Detection** -- `detectMagicDocHeader(content)` scans for the pattern `# MAGIC DOC: [title]` at the start of a file. Optionally detects italicized instructions on the next line.
2. **Registration** -- `registerMagicDoc(filePath)` adds the file to the `trackedMagicDocs` map (idempotent per path).
3. **Listening** -- A `registerFileReadListener()` hook detects Magic Doc headers when files are read during normal tool use.
4. **Update cycle** -- A `registerPostSamplingHook()` fires after each assistant turn. For each tracked doc:
   - Re-reads the file (with a cloned `FileStateCache` to bypass dedup).
   - Verifies the header is still present.
   - Runs a forked `magic-docs` agent (Sonnet model) with `buildMagicDocsUpdatePrompt()`.
   - The agent has access only to the `Edit` tool to modify the doc.
5. **Cleanup** -- `clearTrackedMagicDocs()` resets tracking state.

---

## 8. Prompt Suggestions

**Directory:** `services/PromptSuggestion/`

Generates follow-up prompt suggestions after Claude completes a response, helping users continue their workflow.

### Suggestion generation (`promptSuggestion.ts`)

- **`shouldEnablePromptSuggestion()`** -- Gating logic that checks (in order): env var override, GrowthBook feature flag (`tengu_chomp_inflection`), non-interactive mode (disabled), swarm teammate status (disabled), user settings, and rate limits.
- **`getPromptVariant()`** -- Returns the prompt variant (`'user_intent'`).
- **`generateSuggestion()`** -- Runs a forked agent to generate a suggestion based on conversation context.
- Uses a dedicated abort controller (`currentAbortController`) so suggestions can be cancelled when the user types.

### Speculation (`speculation.ts`)

Speculative execution that pre-runs the suggested prompt in an overlay filesystem:

- **`isSpeculationEnabled()` / `startSpeculation()`** -- Controls speculative pre-execution.
- **Limits:** `MAX_SPECULATION_TURNS = 20`, `MAX_SPECULATION_MESSAGES = 100`.
- **Safety:** Categorizes tools into `WRITE_TOOLS` (Edit, Write, NotebookEdit) and `SAFE_READ_ONLY_TOOLS` (Read, Glob, Grep, ToolSearch, LSP, TaskGet, TaskList). Write operations go to an overlay directory.
- **`safeRemoveOverlay(overlayPath)`** -- Cleans up overlay directories after speculation completes or is cancelled.
- Speculation results are stored as `SpeculationState` / `SpeculationResult` in the app state.

---

## 9. LSP Service

**Directory:** `services/lsp/`

Integrates Language Server Protocol servers to provide diagnostics, definitions, and other IDE-like features.

### Architecture

The LSP subsystem follows a manager-instance pattern:

- **`createLSPServerManager()`** (`LSPServerManager.ts`) -- Factory that creates a manager routing requests based on file extensions. Maintains:
  - `servers` map (server name to instance).
  - `extensionMap` (file extension to server names).
  - `openedFiles` map (URI to owning server name).
  - Methods: `initialize()`, `shutdown()`, `getServerForFile()`, `ensureServerStarted()`, `sendRequest()`, `openFile()`, `changeFile()`, `saveFile()`, `closeFile()`, `isFileOpen()`.

- **`createLSPServerInstance()`** (`LSPServerInstance.ts`) -- Manages a single LSP server lifecycle:
  - State tracking: `LspServerState` (starting, running, stopped, crashed).
  - Health monitoring with crash callback (`onCrash`).
  - Transient error retry: `MAX_RETRIES_FOR_TRANSIENT_ERRORS = 3` with 500ms base exponential backoff for `content modified` errors (`LSP_ERROR_CONTENT_MODIFIED = -32801`).
  - Methods: `start()`, `stop()`, `restart()`, `isHealthy()`, `sendRequest()`, `sendNotification()`, `onNotification()`.

- **`createLSPClient()`** (`LSPClient.ts`) -- Low-level wrapper around `vscode-jsonrpc` `MessageConnection`:
  - Spawns the LSP server process via stdio.
  - Manages connection lifecycle (start, initialize, stop).
  - Supports lazy handler registration (queued before connection is ready).
  - Methods: `start()`, `initialize()`, `sendRequest()`, `sendNotification()`, `onNotification()`, `onRequest()`, `stop()`.

### Configuration

`services/lsp/config.ts` -- LSP servers are only loaded from plugins (not user/project settings):

- **`getAllLspServers()`** -- Loads all enabled plugins and extracts LSP server configurations via `getPluginLspServers()`. Plugin loading errors are tolerated (fail-open).

### Diagnostics

- **`LSPDiagnosticRegistry.ts`** -- Stores asynchronous diagnostics from LSP servers:
  - `registerPendingLSPDiagnostic()` -- Stores diagnostics received via `textDocument/publishDiagnostics`.
  - Cross-turn deduplication via LRU cache (`MAX_DELIVERED_FILES = 500`).
  - Volume limits: `MAX_DIAGNOSTICS_PER_FILE = 10`, `MAX_TOTAL_DIAGNOSTICS = 30`.
  - Diagnostics are delivered as attachments in the next query.

- **`passiveFeedback.ts`** -- Converts LSP diagnostics to Claude's `DiagnosticFile[]` format:
  - `formatDiagnosticsForAttachment()` -- Maps LSP severity (1-4) to Claude severity strings (Error/Warning/Info/Hint).
  - `mapLSPSeverity()` -- The severity mapping function.

---

## 10. OAuth Service

**Directory:** `services/oauth/`

> For complete authentication details, see [auth.md](./auth.md).

Implements OAuth 2.0 Authorization Code flow with PKCE:

- **`OAuthService`** (`index.ts`) -- Main service class. `startOAuthFlow()` supports automatic (browser redirect to localhost) and manual (copy-paste code) flows. Uses `AuthCodeListener` for the localhost callback server.
- **`client.ts`** -- `buildAuthUrl()` constructs authorization URLs for both Console and Claude.ai. `shouldUseClaudeAIAuth()` checks for the `CLAUDE_AI_INFERENCE_SCOPE`. Token exchange, refresh, and profile fetching.
- **`crypto.ts`** -- PKCE utilities: `generateCodeVerifier()`, `generateCodeChallenge()`, `generateState()`.
- **`auth-code-listener.ts`** -- Local HTTP server for receiving OAuth callbacks.
- **`getOauthProfile.ts`** -- Fetches user profile from OAuth tokens.

---

## 11. Plugins Service

**Directory:** `services/plugins/`

> For complete plugin architecture details, see [plugins.md](./plugins.md).

Core plugin operations for install, uninstall, enable, disable, and update:

- **`pluginOperations.ts`** -- Pure library functions (no `process.exit()`, no console output). Used by both CLI commands and interactive UI. Handles marketplace integration, dependency resolution (`findReverseDependents()`), and plugin data directory management.
- **`pluginCliCommands.ts`** -- CLI command implementations for plugin management.
- **`PluginInstallationManager.ts`** -- Manages plugin installation lifecycle.

---

## 12. Key Source Files Reference

| File | Service | Key exports |
|------|---------|-------------|
| `services/mcp/client.ts` | MCP | `ensureConnectedClient()`, `getMcpToolsCommandsAndResources()`, `callMCPToolWithUrlElicitationRetry()`, `processMCPResult()`, `reconnectMcpServerImpl()`, `callIdeRpc()` |
| `services/mcp/config.ts` | MCP | `getAllMcpConfigs()`, `getMcpConfigsByScope()`, `getEnterpriseMcpFilePath()`, `isMcpServerDisabled()` |
| `services/mcp/types.ts` | MCP | `ConfigScope`, `McpStdioServerConfig`, `McpSSEServerConfig`, `McpHTTPServerConfig`, `ScopedMcpServerConfig`, `MCPServerConnection` |
| `services/mcp/MCPConnectionManager.tsx` | MCP | `MCPConnectionManager`, `useMcpReconnect()`, `useMcpToggleEnabled()` |
| `services/mcp/useManageMCPConnections.ts` | MCP | Connection lifecycle, tool/resource/command refresh |
| `services/mcpServerApproval.tsx` | MCP | `handleMcpjsonServerApprovals()` |
| `services/mcp/auth.ts` | MCP | `ClaudeAuthProvider`, `wrapFetchWithStepUpDetection()` |
| `services/analytics/index.ts` | Analytics | `logEvent()`, `logEventAsync()`, `attachAnalyticsSink()`, `stripProtoFields()` |
| `services/analytics/sink.ts` | Analytics | `initializeAnalyticsSink()` |
| `services/analytics/growthbook.ts` | Analytics | `getFeatureValue_CACHED_MAY_BE_STALE()` |
| `services/diagnosticTracking.ts` | Diagnostics | `DiagnosticTrackingService` |
| `cost-tracker.ts` | Cost | `getStoredSessionCosts()`, `restoreCostStateForSession()`, `saveCurrentSessionCosts()`, `formatTotalCost()` |
| `costHook.ts` | Cost | `useCostSummary()` |
| `services/voice.ts` | Voice | `loadAudioNapi()`, `checkRecordingAvailability()`, `probeArecord()` |
| `services/voiceStreamSTT.ts` | Voice | `VoiceStreamConnection`, `VoiceStreamCallbacks`, `isVoiceStreamAvailable()` |
| `services/voiceKeyterms.ts` | Voice | `getVoiceKeyterms()`, `splitIdentifier()` |
| `services/rateLimitMessages.ts` | Rate limits | `getRateLimitMessage()`, `isRateLimitErrorMessage()` |
| `services/rateLimitMocking.ts` | Rate limits | `processRateLimitHeaders()`, `checkMockRateLimitError()` |
| `services/mockRateLimits.ts` | Rate limits | `MockScenario`, mock header generation |
| `services/claudeAiLimits.ts` | Rate limits | `ClaudeAILimits` state management |
| `services/policyLimits/index.ts` | Policy | `initializePolicyLimitsLoadingPromise()`, `loadPolicyLimits()` |
| `services/policyLimits/types.ts` | Policy | `PolicyLimitsResponse`, `PolicyLimitsFetchResult` |
| `services/compact/compact.ts` | Compaction | `compactConversation()`, `partialCompactConversation()`, `buildPostCompactMessages()` |
| `services/compact/autoCompact.ts` | Compaction | `getAutoCompactThreshold()`, `calculateTokenWarningState()`, `getEffectiveContextWindowSize()` |
| `services/compact/microCompact.ts` | Compaction | Time-based/cached micro-compaction, `COMPACTABLE_TOOLS` |
| `services/compact/grouping.ts` | Compaction | `groupMessagesByApiRound()` |
| `services/compact/prompt.ts` | Compaction | Compaction prompt templates |
| `services/MagicDocs/magicDocs.ts` | Magic Docs | `detectMagicDocHeader()`, `registerMagicDoc()`, `clearTrackedMagicDocs()` |
| `services/PromptSuggestion/promptSuggestion.ts` | Suggestions | `shouldEnablePromptSuggestion()`, `generateSuggestion()`, `getPromptVariant()` |
| `services/PromptSuggestion/speculation.ts` | Suggestions | `isSpeculationEnabled()`, `startSpeculation()` |
| `services/lsp/LSPServerManager.ts` | LSP | `createLSPServerManager()` |
| `services/lsp/LSPServerInstance.ts` | LSP | `createLSPServerInstance()` |
| `services/lsp/LSPClient.ts` | LSP | `createLSPClient()` |
| `services/lsp/config.ts` | LSP | `getAllLspServers()` |
| `services/lsp/LSPDiagnosticRegistry.ts` | LSP | `registerPendingLSPDiagnostic()` |
| `services/lsp/passiveFeedback.ts` | LSP | `formatDiagnosticsForAttachment()` |
| `services/oauth/index.ts` | OAuth | `OAuthService` |
| `services/oauth/client.ts` | OAuth | `buildAuthUrl()`, `shouldUseClaudeAIAuth()` |
| `services/plugins/pluginOperations.ts` | Plugins | Plugin install/uninstall/enable/disable operations |
| `services/tokenEstimation.ts` | Tokens | `roughTokenCountEstimation()` |
