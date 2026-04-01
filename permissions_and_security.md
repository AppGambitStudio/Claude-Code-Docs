# Permission & Security System

## 1. Overview

Claude Code's permission system controls which tools the AI assistant can execute and under what conditions. Every tool invocation passes through a multi-layered permission pipeline that evaluates rules, modes, classifiers, and hooks before execution proceeds.

The system is designed around three outcomes for any tool use request:

- **Allow** -- the tool executes immediately.
- **Ask** -- the user is prompted for approval (or a hook/classifier decides).
- **Deny** -- the tool is blocked outright and the model receives an error message.

Key source entry point: `hasPermissionsToUseTool()` in `utils/permissions/permissions.ts:473`.

---

## 2. Permission Modes

Permission modes set the baseline posture for the entire session. They are defined in `types/permissions.ts:16-36`.

### External Modes (user-addressable)

| Mode | Title | Behavior |
|------|-------|----------|
| `default` | Default | Every non-read tool prompts the user for approval. |
| `acceptEdits` | Accept Edits | File write/edit tools in the working directory are auto-allowed; shell commands and other tools still prompt. |
| `plan` | Plan Mode | The assistant discusses plans but does not execute tools. Pauses execution. If the session was started with `bypassPermissions`, bypass remains active when exiting plan mode. |
| `bypassPermissions` | Bypass Permissions | All tools are auto-allowed except explicit deny rules, content-specific ask rules, and safety-check paths (`.git/`, `.claude/`, shell configs). |
| `dontAsk` | Don't Ask | Any tool that would normally prompt (`ask`) is instead silently denied. Useful for non-interactive pipelines. |

### Internal Modes

| Mode | Title | Availability |
|------|-------|-------------|
| `auto` | Auto Mode | Requires the `TRANSCRIPT_CLASSIFIER` feature flag. Uses an AI classifier to decide whether to allow or block tool uses, without prompting the user. |
| `bubble` | Bubble | Internal-only mode for permission delegation to parent contexts. |

### Mode Cycling (Shift+Tab)

Users cycle through modes with Shift+Tab. The cycle order is implemented in `utils/permissions/getNextPermissionMode.ts:34-79`:

```
default -> acceptEdits -> plan -> [bypassPermissions] -> [auto] -> default
```

- `bypassPermissions` appears only when `isBypassPermissionsModeAvailable` is true (session was started with `--dangerously-skip-permissions`).
- `auto` appears only when `TRANSCRIPT_CLASSIFIER` is enabled and `isAutoModeGateEnabled()` returns true.
- For Anthropic internal users, the cycle is shortened: `default -> bypassPermissions -> [auto] -> default` (skipping `acceptEdits` and `plan`).

Mode transitions go through `transitionPermissionMode()` in `utils/permissions/permissionSetup.ts`, which handles context cleanup such as stripping dangerous allow rules when entering auto mode.

### Mode Configuration

Defined in `utils/permissions/PermissionMode.ts:42-91`. Each mode has a `title`, `shortTitle`, `symbol` (for the UI), `color` key, and an `external` mapping (auto maps to `default` externally).

---

## 3. Permission Rules & Sources

### Rule Structure

A permission rule (`PermissionRule` in `types/permissions.ts:75-79`) combines three things:

```typescript
type PermissionRule = {
  source: PermissionRuleSource
  ruleBehavior: PermissionBehavior   // 'allow' | 'deny' | 'ask'
  ruleValue: PermissionRuleValue     // { toolName: string, ruleContent?: string }
}
```

### Rule Sources (Hierarchy)

Rules originate from multiple sources (`types/permissions.ts:54-63`), listed here from most local to most global:

| Source | Location | Mutability |
|--------|----------|-----------|
| `session` | In-memory only (ephemeral) | Read/write |
| `cliArg` | CLI flags (`--permission-mode`, `--allowedTools`) | Read-only at runtime |
| `command` | Programmatic commands | Read-only |
| `localSettings` | `.claude.local.json` (project-local, gitignored) | Read/write |
| `projectSettings` | `.claude/settings.json` (shared with team) | Read/write |
| `userSettings` | `~/.claude/settings.json` (global user prefs) | Read/write |
| `flagSettings` | Feature flags / remote config | Read-only |
| `policySettings` | Organization policies | Read-only |

Rules from all sources are aggregated. The system evaluates deny rules first, then ask rules, then allow rules (see section 4).

### Rule Format and Parsing

Rules are parsed by `permissionRuleValueFromString()` in `utils/permissions/permissionRuleParser.ts:93-133`:

| Format | Example | Meaning |
|--------|---------|---------|
| `ToolName` | `Bash` | Matches all uses of the tool |
| `ToolName(content)` | `Bash(npm install)` | Matches tool use with specific content |
| `ToolName(prefix:*)` | `Bash(npm:*)` | Legacy prefix syntax -- matches commands starting with `npm` |
| `ToolName(pattern *)` | `Bash(git *)` | Wildcard syntax -- `*` matches any characters |
| `mcp__server` | `mcp__server1` | Matches all tools from an MCP server |
| `mcp__server__*` | `mcp__server1__*` | Wildcard for all tools from an MCP server |

Parentheses in rule content are escaped with backslashes. The parser (`permissionRuleParser.ts:55-79`) handles `\(` and `\)` escaping/unescaping. Legacy tool names (e.g., `Task` -> `Agent`, `KillShell` -> `TaskStop`) are normalized via `normalizeLegacyToolName()` at `permissionRuleParser.ts:31`.

### Shell Rule Matching

For shell tools (Bash, PowerShell), command matching is handled by `utils/permissions/shellRuleMatching.ts`. Rules are parsed into a discriminated union (`ShellPermissionRule`):

- **exact** -- literal string match
- **prefix** -- legacy `:*` syntax, e.g., `npm:*` matches commands starting with `npm`
- **wildcard** -- glob-like `*` patterns converted to regex by `matchWildcardPattern()` (`shellRuleMatching.ts:90-153`)

The wildcard matcher converts `*` to `.*` in a regex, supports `\*` for literal asterisks and `\\` for literal backslashes, and uses the `s` (dotAll) flag so wildcards match across newlines. A trailing ` *` pattern (single wildcard) is made optional so `git *` matches both `git add` and bare `git`.

### Dangerous Pattern Stripping

When entering auto mode, the system strips allow rules that would let the model execute arbitrary code. These patterns are defined in `utils/permissions/dangerousPatterns.ts`:

```typescript
// Cross-platform code execution entry points (dangerousPatterns.ts:18-42)
const CROSS_PLATFORM_CODE_EXEC = [
  'python', 'python3', 'node', 'deno', 'ruby', 'perl', 'php',
  'npx', 'bunx', 'npm run', 'yarn run', 'bash', 'sh', 'ssh', ...
]

// Bash-specific dangerous patterns (dangerousPatterns.ts:44-80)
const DANGEROUS_BASH_PATTERNS = [
  ...CROSS_PLATFORM_CODE_EXEC,
  'zsh', 'fish', 'eval', 'exec', 'env', 'xargs', 'sudo', ...
]
```

Stripped rules are stored in `ToolPermissionContext.strippedDangerousRules` so they can be restored when exiting auto mode.

---

## 4. Permission Checking Flow

The full permission check is a pipeline implemented across `hasPermissionsToUseTool()` (outer, `permissions.ts:473`) and `hasPermissionsToUseToolInner()` (core logic, `permissions.ts:1158`).

### Step-by-Step Data Flow

```
Tool Execution Request
        |
        v
hasPermissionsToUseTool()          [permissions.ts:473]
        |
        v
hasPermissionsToUseToolInner()     [permissions.ts:1158]
        |
        |-- Step 1a: Check deny rules (entire tool)
        |     getDenyRuleForTool() -> if match, return DENY
        |
        |-- Step 1b: Check ask rules (entire tool)
        |     getAskRuleForTool() -> if match, return ASK
        |     (exception: Bash with sandbox auto-allow falls through)
        |
        |-- Step 1c: Tool-specific permission check
        |     tool.checkPermissions(parsedInput, context)
        |     Returns PermissionResult (allow/deny/ask/passthrough)
        |
        |-- Step 1d: Tool denied -> return DENY
        |
        |-- Step 1e: Tool requires user interaction -> return ASK
        |
        |-- Step 1f: Content-specific ask rules respected even in bypass mode
        |     (e.g., Bash(npm publish:*) ask rule)
        |
        |-- Step 1g: Safety checks are bypass-immune
        |     (.git/, .claude/, .vscode/, shell configs)
        |
        |-- Step 2a: bypassPermissions mode -> return ALLOW
        |     (also: plan mode with isBypassPermissionsModeAvailable)
        |
        |-- Step 2b: Entire tool in allow rules -> return ALLOW
        |     toolAlwaysAllowedRule()
        |
        |-- Step 3: Convert passthrough to ASK
        |
        v
Back in hasPermissionsToUseTool()  [permissions.ts:473]
        |
        |-- On ALLOW: reset consecutive denial counter, return
        |
        |-- dontAsk mode: convert ASK -> DENY
        |
        |-- auto mode (if ASK):
        |     |-- Non-classifier-approvable safety checks -> prompt/deny
        |     |-- Accept-Edits fast path (re-check with acceptEdits mode)
        |     |-- Safe tool allowlist check (isAutoModeAllowlistedTool)
        |     |-- Run classifier (classifyYoloAction)
        |     |-- On classifier block: track denials, check limits
        |     |-- On classifier allow: return ALLOW
        |
        |-- shouldAvoidPermissionPrompts (headless):
        |     |-- Run PermissionRequest hooks
        |     |-- If no hook decides -> auto-deny
        |
        v
  PermissionDecision returned to caller
        |
        v
  If ASK: interactive permission handler (hooks/toolPermission/)
```

### ToolPermissionContext

The context object (`Tool.ts:123-138`, `types/permissions.ts:427-441`) carries all permission state:

```typescript
type ToolPermissionContext = {
  mode: PermissionMode
  additionalWorkingDirectories: Map<string, AdditionalWorkingDirectory>
  alwaysAllowRules: ToolPermissionRulesBySource
  alwaysDenyRules: ToolPermissionRulesBySource
  alwaysAskRules: ToolPermissionRulesBySource
  isBypassPermissionsModeAvailable: boolean
  strippedDangerousRules?: ToolPermissionRulesBySource
  shouldAvoidPermissionPrompts?: boolean
  awaitAutomatedChecksBeforeDialog?: boolean
  prePlanMode?: PermissionMode
}
```

`ToolPermissionRulesBySource` is a map from `PermissionRuleSource` to `string[]`, where each string is a rule in the `ToolName(content)` format.

---

## 5. Secure Storage

Credentials (OAuth tokens, API keys) are stored using platform-appropriate secure storage. Implementation is in `utils/secureStorage/`.

### Platform Selection (`utils/secureStorage/index.ts`)

```typescript
function getSecureStorage(): SecureStorage {
  if (process.platform === 'darwin') {
    return createFallbackStorage(macOsKeychainStorage, plainTextStorage)
  }
  return plainTextStorage
}
```

- **macOS**: Primary is macOS Keychain via the `security` CLI; falls back to plaintext if Keychain is unavailable (e.g., locked Keychain in SSH sessions).
- **Linux/Windows**: Plaintext storage (libsecret support is planned but not yet implemented).

### macOS Keychain Implementation (`utils/secureStorage/macOsKeychainStorage.ts`)

Operations use the `security` CLI tool:

| Operation | Command |
|-----------|---------|
| Read | `security find-generic-password -a <user> -w -s <service>` |
| Write | `security -i` with `add-generic-password -U -a <user> -s <service> -X <hex>` via stdin |
| Delete | `security delete-generic-password -a <user> -s <service>` |

Key implementation details:

- **Stdin line limit**: The `security -i` stdin buffer is 4096 bytes (`SECURITY_STDIN_LINE_LIMIT = 4096 - 64`). Payloads exceeding this fall back to argv-based invocation (`macOsKeychainStorage.ts:24,121-146`).
- **Data encoding**: JSON is converted to hexadecimal for storage to avoid escaping issues (`macOsKeychainStorage.ts:109`).
- **Service name**: Generated by `getMacOsKeychainStorageServiceName()` in `macOsKeychainHelpers.ts:29-41`, incorporating an OAuth suffix and a hash of non-default config directories for isolation.

### Caching Strategy (`utils/secureStorage/macOsKeychainHelpers.ts:69-85`)

The Keychain cache is a critical performance optimization (each `security` spawn costs ~500ms synchronously):

```typescript
const keychainCacheState = {
  cache: { data: SecureStorageData | null, cachedAt: number },
  generation: number,          // incremented on invalidation
  readInFlight: Promise | null // deduplicates concurrent readAsync() calls
}
```

- **TTL**: 30 seconds (`KEYCHAIN_CACHE_TTL_MS = 30_000`).
- **Stale-while-error**: If a Keychain read fails but a cached value exists, the stale value is served rather than returning null. This prevents transient `security` failures from surfacing as "Not logged in" errors.
- **Generation counter**: `clearKeychainCache()` increments the generation. If `readAsync()` captured an older generation before spawning, its result is discarded to avoid overwriting fresher data.
- **In-flight deduplication**: `readInFlight` ensures that concurrent async reads share a single subprocess spawn.
- **Keychain lock detection**: `isMacOsKeychainLocked()` (`macOsKeychainStorage.ts:211-231`) checks for exit code 36 from `security show-keychain-info`, cached for the process lifetime.

---

## 6. Auto Mode & Classifier

Auto mode replaces user prompts with an AI classifier that evaluates tool use safety. It requires the `TRANSCRIPT_CLASSIFIER` feature flag.

### Three-Stage Fast Path

Before invoking the classifier API, auto mode applies fast paths (`permissions.ts:596-686`):

1. **Safety check guard**: Non-classifier-approvable safety checks (e.g., Windows path bypass attempts, cross-machine bridge messages) force a prompt or deny. Classifier-approvable safety checks (sensitive file paths like `.claude/`, `.git/`) fall through to the classifier.

2. **Accept-Edits fast path**: Re-evaluates the tool with `acceptEdits` mode. If the tool would be allowed in accept-edits mode (e.g., file edits within the working directory), it is auto-allowed without a classifier call. Exceptions: `Agent` and `REPL` tools are excluded because their `checkPermissions` would silently bypass the classifier.

3. **Safe tool allowlist**: `isAutoModeAllowlistedTool()` in `utils/permissions/classifierDecision.ts:56-98` checks against `SAFE_YOLO_ALLOWLISTED_TOOLS`:

   ```
   FileRead, Grep, Glob, LSP, ToolSearch, ListMcpResources,
   ReadMcpResourceTool, TodoWrite, TaskCreate, TaskGet, TaskUpdate,
   TaskList, TaskStop, TaskOutput, AskUserQuestion, EnterPlanMode,
   ExitPlanMode, TeamCreate, TeamDelete, SendMessage, Sleep,
   WorkflowTool (if enabled), YoloClassifier
   ```

### Classifier Invocation

If no fast path applies, the classifier is called via `classifyYoloAction()` from `utils/permissions/yoloClassifier.ts`:

```
formatActionForClassifier(tool.name, input)  ->  classifyYoloAction(messages, action, tools, context, signal)
```

The classifier returns a `YoloClassifierResult` (`types/permissions.ts:346-397`):

```typescript
type YoloClassifierResult = {
  shouldBlock: boolean
  reason: string
  model: string
  stage?: 'fast' | 'thinking'       // 2-stage classifier
  usage?: ClassifierUsage
  durationMs?: number
  stage1Usage?: ClassifierUsage      // token usage from stage 1
  stage2Usage?: ClassifierUsage      // token usage from stage 2
  unavailable?: boolean              // API error
  transcriptTooLong?: boolean        // context window exceeded
  // ... request IDs, message IDs for telemetry join
}
```

### 2-Stage Decision Architecture

The classifier uses a 2-stage approach:

- **Stage 1 (fast)**: Uses `tool_use` structured output for quick decisions.
- **Stage 2 (thinking)**: Uses XML-based output with extended thinking for complex cases.

Stage metadata (`stage1Usage`, `stage1DurationMs`, `stage1RequestId`, `stage2Usage`, etc.) is tracked for overhead analysis.

### Failure Handling

- **Classifier unavailable** (API error): Behavior depends on the `tengu_iron_gate_closed` feature gate (`permissions.ts:846-876`). When enabled (default), the classifier fails closed (denies with retry guidance). When disabled, it fails open (falls back to normal prompting).
- **Transcript too long**: Falls back to normal prompting in interactive mode; aborts in headless mode (transcript only grows, so retrying is futile).
- **Refresh interval**: The iron gate check refreshes every 30 minutes (`CLASSIFIER_FAIL_CLOSED_REFRESH_MS = 30 * 60 * 1000`).

### Denial Tracking (`utils/permissions/denialTracking.ts`)

Auto mode tracks consecutive and total classifier denials:

```typescript
const DENIAL_LIMITS = {
  maxConsecutive: 3,
  maxTotal: 20,
}
```

- `recordDenial()` increments both `consecutiveDenials` and `totalDenials`.
- `recordSuccess()` resets `consecutiveDenials` to 0 (any allowed tool, even rule-based, breaks the streak).
- `shouldFallbackToPrompting()` returns true when either limit is exceeded, at which point the system falls back to interactive prompting so the user can review.

---

## 7. Interactive Permission Handling

When a tool use results in an `ask` decision, the interactive permission handler manages the user prompt and resolves the decision.

### PermissionContext (`hooks/toolPermission/PermissionContext.ts`)

`createPermissionContext()` (`PermissionContext.ts:96`) builds a context object with these key methods:

| Method | Purpose |
|--------|---------|
| `logDecision()` | Records the permission decision to analytics |
| `persistPermissions()` | Writes permission updates to settings files and updates the in-memory context |
| `cancelAndAbort()` | Returns an ASK decision with rejection feedback; aborts the controller if no feedback provided |
| `resolveIfAborted()` | Checks if the abort controller has fired; resolves with cancel if so |
| `runHooks()` | Executes `PermissionRequest` hooks that can allow or deny before the user sees the prompt |
| `tryClassifier()` | (Bash only, `BASH_CLASSIFIER` flag) Awaits an async classifier auto-approval |

### ResolveOnce Pattern (`PermissionContext.ts:63-94`)

Permission resolution uses a `ResolveOnce<T>` guard to handle the race between multiple resolution sources (user input, hooks, classifier, abort):

```typescript
type ResolveOnce<T> = {
  resolve(value: T): void
  isResolved(): boolean
  claim(): boolean  // atomic check-and-mark; returns true if caller won the race
}
```

This prevents double-resolution when, for example, a hook approves at the same time the user clicks Allow.

### Permission Approval Sources

Approvals are tagged with their source (`PermissionContext.ts:45-48`):

- `{ type: 'hook', permanent?: boolean }` -- from a PermissionRequest hook
- `{ type: 'user', permanent: boolean }` -- from user interaction
- `{ type: 'classifier' }` -- from the bash classifier

### Headless / Async Agent Flow

When `shouldAvoidPermissionPrompts` is true (background agents, subagents), the system (`permissions.ts:932-952`):

1. Runs `PermissionRequest` hooks via `runPermissionRequestHooksForHeadlessAgent()`.
2. If a hook allows or denies, that decision is returned.
3. If no hook decides, the tool is auto-denied with a message explaining that permission prompts are unavailable.

Hook results can include `updatedPermissions` for persistent rule changes and an `interrupt` flag to abort the entire agent.

---

## 8. Permission Persistence

### Update Types (`types/permissions.ts:98-131`)

Permission updates are a discriminated union:

| Type | Fields | Effect |
|------|--------|--------|
| `addRules` | `destination`, `rules[]`, `behavior` | Appends rules to the target |
| `replaceRules` | `destination`, `rules[]`, `behavior` | Replaces all rules of the given behavior |
| `removeRules` | `destination`, `rules[]`, `behavior` | Removes specific rules |
| `setMode` | `destination`, `mode` | Changes the permission mode |
| `addDirectories` | `destination`, `directories[]` | Adds working directories to scope |
| `removeDirectories` | `destination`, `directories[]` | Removes working directories from scope |

### Destinations (`types/permissions.ts:88-94`)

| Destination | File | Shared |
|-------------|------|--------|
| `userSettings` | `~/.claude/settings.json` | No (per-user global) |
| `projectSettings` | `.claude/settings.json` | Yes (committed to repo) |
| `localSettings` | `.claude.local.json` | No (gitignored) |
| `session` | In-memory | No (ephemeral) |
| `cliArg` | N/A | No (read-only) |

### Apply and Persist Flow (`utils/permissions/PermissionUpdate.ts`)

1. `applyPermissionUpdate(context, update)` -- returns a new `ToolPermissionContext` with the update applied in-memory.
2. `applyPermissionUpdates(context, updates[])` -- applies multiple updates sequentially.
3. `persistPermissionUpdates(updates[])` -- writes updates to their target settings files on disk.

The `PermissionContext.persistPermissions()` method in the interactive handler calls both persist and apply, then checks `supportsPersistence(destination)` to determine if the rule is durable.

### Rule Deletion (`permissions.ts:1329-1355`)

Rules from `policySettings`, `flagSettings`, and `command` sources cannot be deleted (read-only). Rules from `localSettings`, `userSettings`, and `projectSettings` are deleted both from the in-memory context (via `applyPermissionUpdate` with `removeRules`) and from the settings file on disk.

---

## 9. Key Source Files Reference

| File | Purpose |
|------|---------|
| `types/permissions.ts` | Core type definitions: modes, behaviors, rules, decisions, classifier types, context |
| `utils/permissions/permissions.ts` | Main permission checking pipeline: `hasPermissionsToUseTool()`, `hasPermissionsToUseToolInner()`, rule accessors |
| `utils/permissions/permissionRuleParser.ts` | Rule string parsing (`permissionRuleValueFromString`), serialization, escaping, legacy tool name normalization |
| `utils/permissions/shellRuleMatching.ts` | Shell command matching: `matchWildcardPattern()`, `parsePermissionRule()`, prefix/wildcard/exact discriminated union |
| `utils/permissions/dangerousPatterns.ts` | Dangerous allow-rule patterns stripped in auto mode (`DANGEROUS_BASH_PATTERNS`, `CROSS_PLATFORM_CODE_EXEC`) |
| `utils/permissions/PermissionMode.ts` | Mode metadata (titles, symbols, colors), `permissionModeFromString()`, `toExternalPermissionMode()` |
| `utils/permissions/getNextPermissionMode.ts` | Shift+Tab mode cycling logic: `getNextPermissionMode()`, `cyclePermissionMode()` |
| `utils/permissions/permissionSetup.ts` | Mode transition logic, auto mode gate checks, dangerous rule stripping, safe tool allowlist (`SAFE_YOLO_ALLOWLISTED_TOOLS`) |
| `utils/permissions/classifierDecision.ts` | `isAutoModeAllowlistedTool()`, safe tool allowlist set |
| `utils/permissions/yoloClassifier.ts` | Classifier invocation: `classifyYoloAction()`, `formatActionForClassifier()` |
| `utils/permissions/denialTracking.ts` | Denial counters: `DENIAL_LIMITS`, `recordDenial()`, `recordSuccess()`, `shouldFallbackToPrompting()` |
| `utils/permissions/PermissionUpdate.ts` | In-memory and on-disk persistence: `applyPermissionUpdate()`, `persistPermissionUpdates()` |
| `utils/permissions/PermissionUpdateSchema.ts` | Zod schemas for permission update validation |
| `utils/permissions/permissionsLoader.ts` | Loading rules from settings files, `shouldAllowManagedPermissionRulesOnly()` |
| `utils/permissions/pathValidation.ts` | Path safety checks for file operations |
| `utils/permissions/filesystem.ts` | `toPosixPath()` and filesystem helpers |
| `utils/permissions/bashClassifier.ts` | Bash-specific classifier logic |
| `utils/permissions/autoModeState.ts` | Auto mode activation state: `isAutoModeActive()` |
| `utils/permissions/bypassPermissionsKillswitch.ts` | Remote killswitch for bypass permissions mode |
| `utils/permissions/shadowedRuleDetection.ts` | Detects rules that are shadowed by broader rules |
| `utils/permissions/permissionExplainer.ts` | Human-readable explanations with risk levels (`RiskLevel`, `PermissionExplanation`) |
| `hooks/toolPermission/PermissionContext.ts` | Interactive permission handler: `createPermissionContext()`, `ResolveOnce`, hook execution |
| `hooks/toolPermission/permissionLogging.ts` | Permission decision analytics logging |
| `utils/secureStorage/index.ts` | Platform-aware storage factory: `getSecureStorage()` |
| `utils/secureStorage/macOsKeychainStorage.ts` | macOS Keychain CRUD, stdin/argv fallback, lock detection |
| `utils/secureStorage/macOsKeychainHelpers.ts` | Keychain cache (TTL, generation counter, stale-while-error), service name generation |
| `utils/secureStorage/plainTextStorage.ts` | Plaintext file-based credential storage fallback |
| `utils/secureStorage/fallbackStorage.ts` | `createFallbackStorage()` -- tries primary, falls back to secondary |
| `Tool.ts` | `ToolPermissionContext` type definition (with `DeepImmutable`), `checkPermissions()` interface |
