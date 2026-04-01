# Authentication, State Management, Configuration, and User Hooks

## 1. Overview

Claude Code manages user identity through a multi-provider authentication system, maintains application state via a reactive store, applies layered configuration from multiple sources, supports a hook system for lifecycle customization, and provides a configurable keybinding system for keyboard shortcuts. This document covers each of these systems and how they interrelate.

---

## 2. Authentication System

### 2.1 Auth Flow & Providers

Authentication is orchestrated in `utils/auth.ts`, which determines the active auth source and manages token lifecycle. The system supports multiple auth providers with a clear priority order:

**Auth token source resolution** (`getAuthTokenSource()`):

1. **`ANTHROPIC_AUTH_TOKEN`** env var (skipped in managed OAuth contexts)
2. **`CLAUDE_CODE_OAUTH_TOKEN`** env var (set by Claude Desktop, SSH tunnels, Agent SDK)
3. **OAuth token from file descriptor** (`CLAUDE_CODE_OAUTH_TOKEN_FILE_DESCRIPTOR`), with a disk fallback for CCR subprocesses
4. **`apiKeyHelper`** from settings (a shell command that returns an API key, skipped in managed OAuth contexts)
5. **OAuth tokens from keychain/secure storage** (Claude.ai subscriber flow)
6. **`ANTHROPIC_API_KEY`** env var
7. **API key from macOS Keychain** (via `security find-generic-password`)

The function `isAnthropicAuthEnabled()` gates whether first-party Anthropic auth (OAuth) is active. It returns `false` when:
- `--bare` mode is used (API-key-only)
- A third-party provider is configured (Bedrock, Vertex, Foundry)
- An external API key or auth token is present (and not in a managed OAuth context like CCR or Claude Desktop)

**Managed OAuth context**: When `CLAUDE_CODE_REMOTE=1` or the entrypoint is `claude-desktop`, the system is in a "managed OAuth context." In this mode, user-level settings like `apiKeyHelper` and `ANTHROPIC_API_KEY` are ignored to prevent header mismatches with the auth-injecting proxy.

### 2.2 OAuth Implementation

The OAuth subsystem lives in `services/oauth/` and implements the OAuth 2.0 Authorization Code flow with PKCE.

#### Core Components

| File | Purpose |
|------|---------|
| `services/oauth/index.ts` | `OAuthService` class - orchestrates the full OAuth flow |
| `services/oauth/client.ts` | URL building, token exchange, refresh, profile fetch |
| `services/oauth/crypto.ts` | PKCE code verifier/challenge generation using SHA-256 |
| `services/oauth/auth-code-listener.ts` | Localhost HTTP server to capture browser redirects |
| `services/oauth/getOauthProfile.ts` | Fetches user profile from `/api/oauth/profile` or `/api/claude_cli_profile` |

#### OAuth Flow

1. **`OAuthService.startOAuthFlow()`** is called with an `authURLHandler` callback.
2. A PKCE code verifier and challenge are generated (`crypto.ts`).
3. An `AuthCodeListener` starts a temporary localhost HTTP server on a random port.
4. Two auth URLs are built via `client.buildAuthUrl()`:
   - **Automatic flow URL**: Redirects to `http://localhost:{port}/callback` (browser opens automatically).
   - **Manual flow URL**: Redirects to a page where the user copies an auth code manually.
5. The system races between the automatic browser callback and manual code entry.
6. On receiving the authorization code, `client.exchangeCodeForTokens()` POSTs to the token endpoint.
7. Profile info (subscription type, rate limit tier, billing type) is fetched via `client.fetchProfileInfo()`.
8. Tokens are formatted into an `OAuthTokens` object containing `accessToken`, `refreshToken`, `expiresAt`, `scopes`, `subscriptionType`, `rateLimitTier`, and `profile`.

#### Token Refresh

`client.refreshOAuthToken()` handles token refresh with several optimizations:
- Skips the profile round-trip (`/api/oauth/profile`) when the global config already has billing type, account creation date, and subscription data. This saves approximately 7 million requests per day fleet-wide.
- Updates display name, billing type, and extra usage settings in global config when profile data changes.
- Scope expansion is supported on refresh (backend allows it via `ALLOWED_SCOPE_EXPANSIONS`).

#### Token Expiry

`isOAuthTokenExpired()` checks if a token will expire within a 5-minute buffer window, ensuring tokens are refreshed proactively.

#### Subscription Types

Derived from `organization.organization_type` in the profile response:
- `claude_max` -> `'max'`
- `claude_pro` -> `'pro'`
- `claude_enterprise` -> `'enterprise'`
- `claude_team` -> `'team'`

### 2.3 API Key Management

API keys are stored and retrieved through several mechanisms:

- **macOS Keychain**: Primary secure storage on macOS, accessed via `utils/secureStorage/`.
- **File descriptor**: For CCR (Claude Code Remote) and managed contexts, tokens can be passed via file descriptors.
- **Environment variables**: `ANTHROPIC_API_KEY`, `ANTHROPIC_AUTH_TOKEN`, `CLAUDE_CODE_OAUTH_TOKEN`.
- **`apiKeyHelper`**: A shell command configured in settings that returns an API key. Cached with a 5-minute TTL (`DEFAULT_API_KEY_HELPER_TTL`).

The `saveApiKey()` and `removeApiKey()` functions handle persistence. `normalizeApiKeyForConfig()` and `maybeRemoveApiKeyFromMacOSKeychainThrows()` (from `utils/authPortable.ts`) handle platform-specific key management.

### 2.4 Login/Logout Commands

#### `/login` Command (`commands/login/`)

- Defined in `commands/login/index.ts` as a `local-jsx` command.
- Can be disabled via `DISABLE_LOGIN_COMMAND` env var.
- Renders a `Login` component that wraps `ConsoleOAuthFlow` in a `Dialog`.
- On successful login, triggers a cascade of post-login actions:
  - Resets cost state for the new account.
  - Refreshes remote managed settings (non-blocking).
  - Refreshes policy limits (non-blocking).
  - Clears user data cache and refreshes GrowthBook feature flags.
  - Enrolls as a trusted device for Remote Control.
  - Resets and re-runs bypass-permissions killswitch checks.
  - Increments `authVersion` in AppState to trigger re-fetching of auth-dependent data.

#### `/logout` Command (`commands/logout/`)

- Defined in `commands/logout/index.ts` as a `local-jsx` command.
- Can be disabled via `DISABLE_LOGOUT_COMMAND` env var.
- `performLogout()` executes the logout sequence:
  1. Flushes telemetry **before** clearing credentials (prevents org data leakage).
  2. Removes the API key.
  3. Wipes all secure storage data.
  4. Clears auth-related caches (OAuth tokens, trusted device tokens, betas, tool schemas, user cache, GrowthBook, Grove config, remote managed settings, policy limits).
  5. Clears `oauthAccount` from global config.
  6. Optionally resets onboarding state.
- After logout, triggers a graceful shutdown with a 200ms delay.

---

## 3. State Management

### 3.1 AppState Architecture

The application state is defined in `state/AppStateStore.ts` as the `AppState` type. It is a deeply immutable object (via `DeepImmutable<>`) with some exceptions for mutable structures like `tasks` and `mcp`.

**Key state fields include:**

| Field | Type | Purpose |
|-------|------|---------|
| `settings` | `SettingsJson` | Current effective settings |
| `verbose` | `boolean` | Verbose output mode |
| `mainLoopModel` | `ModelSetting` | Active model for the main loop |
| `toolPermissionContext` | `ToolPermissionContext` | Permission tracking for tools |
| `agent` | `string \| undefined` | Active agent name from `--agent` flag |
| `kairosEnabled` | `boolean` | Whether assistant mode is enabled |
| `tasks` | `Record<string, TaskState>` | Active background tasks |
| `mcp` | `{ clients, tools, commands, resources }` | MCP server connections and tools |
| `plugins` | `{ enabled, disabled, commands, errors }` | Plugin state |
| `replBridgeEnabled` | `boolean` | Whether the REPL bridge is active |
| `remoteConnectionStatus` | `string` | WebSocket connection state for remote sessions |

The state also tracks UI concerns like `expandedView`, `footerSelection`, `coordinatorTaskIndex`, and companion/buddy reactions.

### 3.2 AppStateStore

The store is implemented in `state/store.ts` as a minimal reactive store:

```typescript
type Store<T> = {
  getState: () => T
  setState: (updater: (prev: T) => T) => void
  subscribe: (listener: Listener) => () => void
}
```

Key characteristics:
- **Immutable updates**: `setState` takes an updater function `(prev: T) => T`. If the updater returns the same reference (`Object.is`), no update occurs.
- **Listener notification**: All subscribed listeners are notified synchronously after state changes.
- **onChange callback**: An optional `onChange` handler receives both `newState` and `oldState` for side effects.

### 3.3 State Access (AppState.tsx)

The React integration is provided through `state/AppState.tsx`:

- **`AppStateProvider`**: Context provider that wraps the app. Cannot be nested (throws if a parent provider already exists). Accepts `initialState` and an `onChangeAppState` callback.
- **`useAppState(selector)`**: Primary hook for reading state. Takes a selector function `(state: AppState) => T` and returns the selected value. Uses `useSyncExternalStore` for React 18 concurrent-mode compatibility. Enforces that selectors do not return the entire state object (must return a derived property for optimized rendering).
- **`useAppStateStore()`**: Returns the raw store for non-React code that needs `getState()` / `setState()` directly.
- **`useAppStateMaybeOutsideOfProvider(selector)`**: Safe version that returns `undefined` when called outside of `AppStateProvider` (useful for components that may render in both contexts).

### 3.4 State Selectors

Selectors in `state/selectors.ts` provide pure data extraction functions:

- **`getViewedTeammateTask(appState)`**: Returns the `InProcessTeammateTaskState` being viewed, or `undefined` if no teammate is selected or the task is not an in-process teammate.
- **`getActiveAgentForInput(appState)`**: Determines where user input should be routed. Returns a discriminated union:
  - `{ type: 'leader' }` - Input goes to the leader agent.
  - `{ type: 'viewed', task }` - Input goes to the viewed in-process teammate.
  - `{ type: 'named_agent', task }` - Input goes to a named local agent.

---

## 4. Configuration System

### 4.1 Settings Hierarchy

Settings are loaded from multiple sources with a defined precedence order (later sources override earlier ones):

| Priority | Source | Constant | File Path |
|----------|--------|----------|-----------|
| 1 (lowest) | User settings | `userSettings` | `~/.claude/settings.json` |
| 2 | Project settings (shared) | `projectSettings` | `<project>/.claude/settings.json` |
| 3 | Local settings (gitignored) | `localSettings` | `<project>/.claude/settings.local.json` |
| 4 | Flag settings (CLI `--settings`) | `flagSettings` | Path from `--settings` flag |
| 5 (highest) | Policy/managed settings | `policySettings` | Platform-managed or remote |

Defined in `utils/settings/constants.ts` as the `SETTING_SOURCES` array. The `getEnabledSettingSources()` function always includes `policySettings` and `flagSettings` regardless of any source restrictions.

**Editable sources** (where users can save permission rules and hooks): `localSettings`, `projectSettings`, `userSettings`. Policy and flag settings are read-only.

#### Settings Loading Pipeline

1. **`getSettingsWithErrors()`** is the main entry point, returning both effective settings and validation errors.
2. Each source is loaded via **`getSettingsForSource(source)`**, which reads and caches per-source JSON.
3. Files are parsed by **`parseSettingsFile(path)`** with Zod schema validation (`SettingsSchema`).
4. Invalid permission rules are filtered before schema validation so one bad rule does not reject the entire file.
5. Sources are merged using `lodash mergeWith` with a custom merge strategy.
6. Results are cached at multiple levels (per-file, per-source, session-level).

#### Settings Schema

The `SettingsSchema` (in `utils/settings/types.ts`) validates settings structure using Zod. Key sections include:
- **`permissions`**: `allow`, `deny`, `ask` arrays of permission rules, plus `defaultMode` and `additionalDirectories`.
- **`env`**: Environment variables (`Record<string, string>`).
- **`hooks`**: Hook definitions keyed by event name.
- **MCP server configuration**: Including allowlists for enterprise.

#### Platform-Specific Managed Settings

Policy settings support multiple delivery mechanisms:
- **Remote managed settings** (fetched from API, see section 4.2)
- **MDM/HKLM/plist** settings (`utils/settings/mdm/`) for enterprise device management
- **File-based managed settings** (`managed-settings.json` + drop-in directory `managed-settings.d/*.json`)

For `policySettings`, the first non-empty source wins in this order: remote > MDM/plist > file > HKCU.

File-based managed settings use a systemd/sudoers-style drop-in convention: the base file (`managed-settings.json`) provides defaults, and drop-in files (sorted alphabetically) override on top.

### 4.2 Remote Managed Settings

Implemented in `services/remoteManagedSettings/`, this system fetches enterprise settings from the Anthropic API.

#### Eligibility (`syncCache.ts`)

Not all users qualify for remote settings. Eligibility is determined by `isRemoteManagedSettingsEligible()`:
- **Console users** (API key): All eligible.
- **OAuth users with enterprise/team subscription**: Eligible.
- **OAuth users with `subscriptionType === null`** (externally-injected tokens): Eligible (the API returns empty for ineligible orgs).
- **Third-party provider users** (Bedrock/Vertex): Not eligible.
- **Custom base URL users**: Not eligible.
- **Cowork/local-agent entrypoint**: Not eligible.

#### Fetch & Cache Flow (`index.ts`)

1. **`initializeRemoteManagedSettingsLoadingPromise()`** creates a promise other systems can await (with a 30-second timeout to prevent deadlocks).
2. **`loadRemoteManagedSettings()`** is called during CLI initialization:
   - Applies cached settings from disk immediately (saves ~77ms on startup).
   - Fetches fresh settings with retry logic (up to 5 retries with exponential backoff).
   - Supports ETag-based HTTP caching (`If-None-Match` header with checksum).
   - Validates response against `RemoteManagedSettingsResponseSchema` and `SettingsSchema`.
   - Performs a security check before applying new settings (`securityCheck.tsx`).
   - Saves to disk and triggers hot-reload via `settingsChangeDetector.notifyChange('policySettings')`.
3. **Background polling** runs every hour to pick up mid-session changes.
4. **`refreshRemoteManagedSettings()`** is called on login/logout to re-fetch with new credentials.

#### Response Format (`types.ts`)

```typescript
{
  uuid: string       // Settings UUID
  checksum: string   // SHA-256 checksum for HTTP caching
  settings: SettingsJson
}
```

Checksums are computed client-side using `computeChecksumFromSettings()`, which normalizes JSON to match Python's `json.dumps(sort_keys=True, separators=(",", ":"))`.

### 4.3 Dynamic Configuration

The global configuration (`utils/config.ts`) manages persistent user state outside of settings:

- **`GlobalConfig`** type: Stores per-user data like `numStartups`, `theme`, `hasCompletedOnboarding`, `oauthAccount`, `userID`, `autoUpdates`, and per-project configs.
- **`ProjectConfig`** type: Per-project data including `allowedTools`, `mcpServers`, `hasTrustDialogAccepted`, MCP server approval fields, and worktree session management.
- **`AccountInfo`** type: OAuth account details (`accountUuid`, `emailAddress`, `organizationUuid`, `displayName`, `billingType`, etc.).
- **`getGlobalConfig()` / `saveGlobalConfig(updater)`**: Read/write the global config file (`~/.claude/config.json`). Uses a re-entrancy guard to prevent infinite recursion when analytics code reads config.

Settings hot-reload is managed by `utils/settings/changeDetector.ts`, which notifies listeners when settings change from any source.

---

## 5. User Hooks System

### 5.1 Hook Types & Definitions

Hooks allow users to run custom logic at specific points in the Claude Code lifecycle. They are configured in the `hooks` section of any settings file.

#### Hook Events

Defined in `entrypoints/sdk/coreSchemas.ts`, the complete list of hook events:

| Event | When It Fires |
|-------|--------------|
| `PreToolUse` | Before a tool is executed |
| `PostToolUse` | After a tool completes successfully |
| `PostToolUseFailure` | After a tool fails |
| `Notification` | When a notification is generated |
| `UserPromptSubmit` | When the user submits a prompt |
| `SessionStart` | When a session begins |
| `SessionEnd` | When a session ends |
| `Stop` | When the agent stops |
| `StopFailure` | When a stop attempt fails |
| `SubagentStart` | When a sub-agent starts |
| `SubagentStop` | When a sub-agent stops |
| `PreCompact` | Before context compaction |
| `PostCompact` | After context compaction |
| `PermissionRequest` | When a permission is requested |
| `PermissionDenied` | When a permission is denied |
| `Setup` | During initial setup |
| `TeammateIdle` | When a teammate agent becomes idle |
| `TaskCreated` | When a background task is created |
| `TaskCompleted` | When a background task completes |
| `Elicitation` | When an elicitation is requested |
| `ElicitationResult` | When an elicitation result is received |
| `ConfigChange` | When configuration changes |
| `WorktreeCreate` | When a git worktree is created |
| `WorktreeRemove` | When a git worktree is removed |
| `InstructionsLoaded` | When CLAUDE.md instructions are loaded |
| `CwdChanged` | When the working directory changes |
| `FileChanged` | When a watched file changes |

#### Hook Command Types (`schemas/hooks.ts`)

Each hook event can have one or more hook commands of different types:

**`command` (BashCommandHook)**: Runs a shell command.
- `command`: Shell command string.
- `shell`: `'bash'` (default, uses `$SHELL`) or `'powershell'` (uses `pwsh`).
- `timeout`: Per-command timeout in seconds.
- `if`: Permission rule syntax filter (e.g., `"Bash(git *)"`) to conditionally run.
- `once`: If true, runs once and is removed.
- `async`: If true, runs in background without blocking.
- `asyncRewake`: Runs in background; wakes the model on exit code 2.
- `statusMessage`: Custom spinner text while running.

**`prompt` (PromptHook)**: Evaluates a prompt with an LLM.
- `prompt`: The prompt text. Use `$ARGUMENTS` for hook input JSON.
- `model`: Optional model override (e.g., `"claude-sonnet-4-6"`).
- Other fields: `if`, `timeout`, `statusMessage`, `once`.

**`http` (HttpHook)**: Sends an HTTP POST request.
- `url`: Endpoint URL.
- `headers`: Additional headers (supports env var interpolation via `$VAR_NAME`).
- `allowedEnvVars`: Explicit allowlist for env var interpolation in headers.
- Other fields: `if`, `timeout`, `statusMessage`, `once`.

**`agent` (AgentHook)**: Runs an agentic verifier.
- `prompt`: Describes what to verify (e.g., `"Verify that unit tests ran and passed."`).
- `model`: Optional model override (defaults to Haiku).
- Other fields: `if`, `timeout`, `statusMessage`, `once`.

### 5.2 Hook Registration & Execution

Hooks are defined in settings files under the `hooks` key, keyed by event name. Each event maps to an array of hook commands:

```json
{
  "hooks": {
    "PreToolUse": [
      { "type": "command", "command": "echo 'Before tool use'" }
    ],
    "SessionStart": [
      { "type": "command", "command": "./setup.sh" }
    ]
  }
}
```

#### Hook Matchers

Hook commands can include an `if` field using permission rule syntax to filter execution. For example, `"if": "Bash(git *)"` only runs the hook when the tool is `Bash` and the command starts with `git`.

#### Callback Hooks (`types/hooks.ts`)

In addition to settings-defined hooks, the system supports programmatic `HookCallback` objects:

```typescript
type HookCallback = {
  type: 'callback'
  callback: (input: HookInput, toolUseID: string | null, abort: AbortSignal | undefined, hookIndex?: number, context?: HookCallbackContext) => Promise<HookJSONOutput>
  timeout?: number
  internal?: boolean  // Excluded from metrics
}
```

These are registered via `HookCallbackMatcher` objects with optional `matcher` strings and `pluginName`.

### 5.3 Hook Responses

Hooks communicate results through JSON output. Two response types exist:

**Sync response** (`SyncHookJSONOutput`):
- `continue`: Whether Claude should continue (default: `true`).
- `suppressOutput`: Hide stdout from transcript.
- `stopReason`: Message shown when `continue` is `false`.
- `decision`: `'approve'` or `'block'`.
- `reason`: Explanation for the decision.
- `systemMessage`: Warning message shown to the user.
- `hookSpecificOutput`: Event-specific data (varies by hook event). Examples:
  - `PreToolUse`: Can override `permissionDecision`, `updatedInput`, `additionalContext`.
  - `SessionStart`: Can set `initialUserMessage`, `watchPaths`.
  - `PermissionRequest`: Returns `allow` (with optional `updatedInput`, `updatedPermissions`) or `deny` (with optional `message`, `interrupt`).

**Async response** (`AsyncHookJSONOutput`):
- `async: true`: Signals the hook is running in the background.
- `asyncTimeout`: Optional timeout for the background operation.

#### Aggregated Results

`AggregatedHookResult` combines results from multiple hooks for the same event, collecting `blockingErrors`, `additionalContexts`, and merging permission decisions.

---

## 6. Keybindings

### 6.1 Keybinding System Architecture

The keybinding system provides a configurable, context-aware keyboard shortcut mechanism built on top of Ink's input handling.

#### Core Components

| File | Purpose |
|------|---------|
| `keybindings/schema.ts` | Zod schemas and action/context definitions |
| `keybindings/defaultBindings.ts` | Built-in default shortcuts |
| `keybindings/loadUserBindings.ts` | User config loader with hot-reload |
| `keybindings/resolver.ts` | Key-to-action resolution engine |
| `keybindings/parser.ts` | Keystroke string parser |
| `keybindings/match.ts` | Key matching with modifier support |
| `keybindings/KeybindingContext.tsx` | React context provider |
| `keybindings/useKeybinding.ts` | React hooks for binding actions |
| `keybindings/validate.ts` | Binding validation and duplicate detection |
| `keybindings/reservedShortcuts.ts` | Non-rebindable shortcuts (ctrl+c, ctrl+d) |

#### Contexts

Keybindings are scoped to UI contexts. Each context activates when its corresponding UI element has focus:

- **`Global`**: Active everywhere.
- **`Chat`**: When the chat input is focused.
- **`Autocomplete`**: When autocomplete menu is visible.
- **`Confirmation`**: When a confirmation/permission dialog is shown.
- **`Transcript`**: When viewing the transcript.
- **`HistorySearch`**: During command history search (ctrl+r).
- **`Task`**: When a task/agent runs in the foreground.
- **`Settings`**: When the settings menu is open.
- **`Tabs`**: During tab navigation.
- **`Attachments`**, **`Footer`**, **`MessageSelector`**, **`DiffDialog`**, **`ModelPicker`**, **`Select`**, **`Plugin`**, **`ThemePicker`**, **`Help`**: Specialized UI contexts.

#### Actions

Actions follow a `namespace:action` naming convention (e.g., `app:interrupt`, `chat:submit`, `history:search`). The full list is defined in `KEYBINDING_ACTIONS` in `keybindings/schema.ts`. Actions can also reference slash commands via the `command:` prefix (e.g., `command:help`).

#### Resolution

The `resolveKey()` function matches input against bindings filtered by active contexts. The last matching binding wins (user overrides take precedence over defaults).

**Chord support**: The `resolveKeyWithChordState()` function handles multi-keystroke sequences (e.g., `ctrl+x ctrl+k` for `chat:killAgents`). When the first keystroke of a chord is pressed, the resolver returns `chord_started` with pending state. Subsequent keystrokes complete or cancel the chord.

#### Default Bindings

Platform-specific defaults are defined in `keybindings/defaultBindings.ts`:
- **Mode cycling**: `shift+tab` on most platforms; `meta+m` on Windows without VT mode support.
- **Image paste**: `ctrl+v` on most platforms; `alt+v` on Windows.
- **Reserved shortcuts**: `ctrl+c` (interrupt) and `ctrl+d` (exit) are defined but cannot be rebound by users.

Notable default bindings:
| Shortcut | Action | Context |
|----------|--------|---------|
| `ctrl+c` | `app:interrupt` | Global |
| `ctrl+d` | `app:exit` | Global |
| `ctrl+l` | `app:redraw` | Global |
| `ctrl+r` | `history:search` | Global |
| `enter` | `chat:submit` | Chat |
| `escape` | `chat:cancel` | Chat |
| `ctrl+x ctrl+k` | `chat:killAgents` | Chat |
| `ctrl+x ctrl+e` | `chat:externalEditor` | Chat |
| `tab` | `autocomplete:accept` | Autocomplete |

### 6.2 Customization

User keybindings are stored in `~/.claude/keybindings.json` and loaded by `keybindings/loadUserBindings.ts`.

#### File Format

```json
{
  "$schema": "https://json.schemastore.org/claude-code-keybindings.json",
  "bindings": [
    {
      "context": "Chat",
      "bindings": {
        "ctrl+enter": "chat:submit",
        "enter": "chat:newline"
      }
    }
  ]
}
```

#### Loading & Hot-Reload

1. Default bindings are parsed first.
2. User bindings are loaded from `~/.claude/keybindings.json` and appended after defaults (last binding wins for overrides).
3. A `chokidar` file watcher monitors the keybindings file for changes, with a 500ms stability threshold.
4. On change, bindings are reloaded and all subscribers are notified via a signal.
5. Validation runs on user config, checking for duplicate keys in raw JSON, invalid contexts, and unknown actions.

#### Feature Gate

Keybinding customization is gated behind the `tengu_keybinding_customization_release` GrowthBook feature flag. When disabled, only default bindings are used.

#### React Integration

- **`KeybindingContext.tsx`**: Provides the binding resolution context to the React tree. Manages active contexts, pending chord state, and handler registration.
- **`useKeybinding(action, handler, options)`**: Hook for a single action. Registers the handler and resolves keystrokes via the context.
- **`useKeybindings(handlers, options)`**: Hook for multiple actions in one call (reduces `useInput` registrations). Handlers returning `false` signal "not consumed," allowing event propagation.

Both hooks use `stopImmediatePropagation()` to prevent other handlers from firing once a binding is matched.

---

## 7. Key Source Files Reference

| File Path | Description |
|-----------|-------------|
| `utils/auth.ts` | Central auth orchestration: token source resolution, API key retrieval, auth state checks |
| `services/oauth/index.ts` | `OAuthService` class implementing OAuth 2.0 + PKCE flow |
| `services/oauth/client.ts` | OAuth URL building, token exchange/refresh, profile fetching, API key creation |
| `services/oauth/crypto.ts` | PKCE cryptographic utilities (code verifier, challenge, state) |
| `services/oauth/auth-code-listener.ts` | `AuthCodeListener` - localhost HTTP server for OAuth redirects |
| `services/oauth/getOauthProfile.ts` | Profile fetching via API key or OAuth token |
| `commands/login/login.tsx` | `/login` command - OAuth login UI with post-login refresh cascade |
| `commands/login/index.ts` | Login command registration |
| `commands/logout/logout.tsx` | `/logout` command - credential cleanup and cache clearing |
| `commands/logout/index.ts` | Logout command registration |
| `state/AppStateStore.ts` | `AppState` type definition with all state fields |
| `state/AppState.tsx` | React context provider, `useAppState` hook, store access |
| `state/store.ts` | Generic reactive store implementation |
| `state/selectors.ts` | Pure selector functions for derived state |
| `utils/settings/settings.ts` | Settings loading, merging, file path resolution, caching |
| `utils/settings/types.ts` | `SettingsSchema` Zod schema, `SettingsJson` type |
| `utils/settings/constants.ts` | `SETTING_SOURCES` array, source names, editable source types |
| `utils/settings/managedPath.ts` | Platform-specific managed settings file paths |
| `utils/settings/changeDetector.ts` | Hot-reload notification for settings changes |
| `utils/settings/mdm/settings.ts` | MDM/HKLM/plist managed settings loader |
| `services/remoteManagedSettings/index.ts` | Remote settings fetch, cache, polling, security checks |
| `services/remoteManagedSettings/syncCache.ts` | Eligibility checks and cache management |
| `services/remoteManagedSettings/types.ts` | `RemoteManagedSettingsResponseSchema`, fetch result types |
| `utils/config.ts` | `GlobalConfig`/`ProjectConfig` types, `getGlobalConfig`/`saveGlobalConfig` |
| `types/hooks.ts` | Hook type definitions, JSON output schemas, result types |
| `schemas/hooks.ts` | Hook command Zod schemas (command, prompt, http, agent) |
| `entrypoints/sdk/coreSchemas.ts` | `HOOK_EVENTS` list, `HookEventSchema`, `BaseHookInputSchema` |
| `keybindings/schema.ts` | `KEYBINDING_CONTEXTS`, `KEYBINDING_ACTIONS`, Zod schemas |
| `keybindings/defaultBindings.ts` | Platform-aware default keybinding definitions |
| `keybindings/loadUserBindings.ts` | User keybinding loader with chokidar hot-reload |
| `keybindings/resolver.ts` | Key-to-action resolution with chord support |
| `keybindings/KeybindingContext.tsx` | React context for keybinding resolution |
| `keybindings/useKeybinding.ts` | `useKeybinding` and `useKeybindings` React hooks |
