# CLI & Terminal Interface

The user-facing terminal experience of Claude Code is a hybrid between standard CLI argument processing and a persistent React component tree rendered in the terminal via a forked Ink framework. This document covers entrypoints, the main loop, the React terminal UI, interactive helpers, the dialog system, the slash command system, and the output pipeline.

---

## 1. Entrypoints (`entrypoints/` and `cli/`)

Claude Code has multiple entry modes, each with its own bootstrap path. The top-level entrypoint is `entrypoints/cli.tsx`, which performs fast-path dispatch before loading the full application.

### Entry Modes

| Mode | Trigger | Bootstrap Path |
|------|---------|---------------|
| **Interactive CLI** | `claude` (no flags, or with `--resume`, `--continue`) | Full `main()` in `main.tsx` via Commander |
| **Headless / Print** | `claude -p "prompt"` | `runHeadless()` in `cli/print.ts` |
| **MCP Server** | `claude mcp serve` | `startMCPServer()` in `entrypoints/mcp.ts` |
| **Bridge / Remote Control** | `claude remote-control` (or `rc`, `bridge`, `sync`) | `bridgeMain()` in `bridge/bridgeMain.ts` |
| **Assistant / Daemon** | `claude daemon [subcommand]` | `daemonMain()` in `daemon/main.ts` |
| **Daemon Worker** | `--daemon-worker=<kind>` (internal) | `runDaemonWorker()` in `daemon/workerRegistry.ts` |
| **Background Sessions** | `claude ps`, `logs`, `attach`, `kill`, `--bg` | Handlers in `cli/bg.ts` |
| **SDK / Structured** | `--output-format stream-json --input-format stream-json` | `StructuredIO` class in `cli/structuredIO.ts` |
| **Direct Connect** | `cc://` or `cc+unix://` URL in argv | `createDirectConnectSession()` in `server/createDirectConnectSession.ts` |
| **Chrome Native Host** | `--chrome-native-host` | `runChromeNativeHost()` in `utils/claudeInChrome/chromeNativeHost.ts` |
| **Environment Runner** | `claude environment-runner` | `environmentRunnerMain()` (BYOC headless runner) |
| **Template Jobs** | `claude new`, `list`, `reply` | `templatesMain()` in `cli/handlers/templateJobs.ts` |

### CLI Argument Parsing

`entrypoints/cli.tsx` provides fast-path exits that avoid loading the full module graph:

- **`--version` / `-v`**: Prints `MACRO.VERSION` and exits immediately (zero imports).
- **`--dump-system-prompt`**: Outputs the rendered system prompt and exits (internal only).
- **`--claude-in-chrome-mcp`**: Starts the Chrome MCP server.

All other paths load the startup profiler (`profileCheckpoint('cli_entry')`) and then dispatch based on `args[0]`. The main interactive/headless path eventually calls `main()` from `main.tsx`, which uses Commander (`@commander-js/extra-typings`) to parse the full option set.

Key CLI options parsed by Commander in `main()`:

| Flag | Purpose |
|------|---------|
| `-p, --print` | Non-interactive headless mode |
| `-c, --continue` | Resume most recent conversation |
| `-r, --resume [id]` | Resume by session ID or open picker |
| `--model <model>` | Override model (alias like `opus` or full ID) |
| `--permission-mode <mode>` | Set permission mode (`default`, `plan`, `auto`, `bypassPermissions`) |
| `--dangerously-skip-permissions` | Bypass all permission checks |
| `--output-format <format>` | `text`, `json`, or `stream-json` |
| `--input-format <format>` | `text` or `stream-json` |
| `--mcp-config <configs...>` | Load MCP server configurations |
| `--system-prompt <prompt>` | Override system prompt |
| `--append-system-prompt <prompt>` | Append to default system prompt |
| `--allowed-tools <tools...>` | Restrict available tools |
| `--disallowed-tools <tools...>` | Deny specific tools |
| `--add-dir <dirs...>` | Additional working directories |
| `--bare` | Minimal mode: skip hooks, LSP, plugins, auto-memory |
| `--max-turns <n>` | Limit agentic turns (headless only) |
| `--max-budget-usd <amount>` | Spending cap (headless only) |
| `--session-id <uuid>` | Use specific session ID |
| `-n, --name <name>` | Display name for the session |
| `--effort <level>` | Effort level (`low`, `medium`, `high`, `max`) |
| `--settings <file-or-json>` | Additional settings source |
| `--json-schema <schema>` | Structured output validation |

### Initialization Flow (`entrypoints/init.ts`)

The `init()` function (memoized, runs once) handles:

1. `enableConfigs()` -- validates and enables the configuration system
2. `applySafeConfigEnvironmentVariables()` -- applies non-dangerous env vars before trust
3. `applyExtraCACertsFromConfig()` -- loads custom CA certs before first TLS handshake
4. `setupGracefulShutdown()` -- registers exit handlers
5. Initialize 1P event logging and OpenTelemetry (deferred)
6. `configureGlobalAgents()` -- proxy setup
7. `configureGlobalMTLS()` -- mutual TLS
8. Pre-connect to Anthropic API
9. Initialize policy limits and remote managed settings loading

---

## 2. The Main Loop (`main.tsx`)

`main.tsx` is the largest file in the codebase. Its `main()` function orchestrates the full lifecycle from process start to interactive session.

### Startup Sequence

```
profileCheckpoint('main_tsx_entry')
  -> startMdmRawRead()           // Fire MDM subprocess reads in parallel
  -> startKeychainPrefetch()     // Pre-read OAuth + legacy API key from keychain
  -> import chain (~135ms)       // Heavy module evaluation
  -> profileCheckpoint('main_tsx_imports_loaded')
  -> main()                      // Commander parses argv
    -> init()                    // Core initialization (configs, env, TLS, analytics)
    -> initializeGrowthBook()    // Feature flags
    -> getRenderContext()        // Create Ink root
    -> showSetupScreens()        // Onboarding, trust dialog, MCP approvals
    -> launchRepl()              // Mount App + REPL into Ink root
```

### Side-Effect Imports at Top of File

The first three imports execute side-effects before the remaining module graph loads:

1. **`startupProfiler.js`** -- `profileCheckpoint('main_tsx_entry')` marks entry time
2. **`mdm/rawRead.js`** -- `startMdmRawRead()` fires MDM subprocesses (`plutil`/`reg query`) in parallel with imports
3. **`keychainPrefetch.js`** -- `startKeychainPrefetch()` reads both macOS keychain entries concurrently (~65ms savings)

### Key Functions

- **`main()`** (line ~585): Top-level entry. Sets up signal handlers, parses deep link URIs, handles assistant mode, then enters Commander's `.action()` callback.
- **`startDeferredPrefetches()`** (line ~388): Fires low-priority background work after the REPL renders (GrowthBook refresh, referral eligibility, MCP URL prefetch).
- **`logManagedSettings()`**: Logs managed settings keys to analytics after `init()`.
- **`logSessionTelemetry()`**: Logs skill and plugin telemetry for the session.

### Headless vs Interactive Split

Inside Commander's `.action()` callback, the flow diverges:

- **Headless (`-p`)**: Calls `runHeadless()` from `cli/print.ts`. No Ink root is created. Output goes directly to stdout.
- **Interactive**: Creates an Ink root via `getRenderContext()`, runs setup screens, then calls `launchRepl()`.

---

## 3. React Terminal UI (`ink/` and `components/`)

Claude Code replaces standard stdout printing with a React application running in the terminal via a heavily customized fork of the Ink framework.

### Ink Framework (`ink/`)

The `ink/` directory contains a forked and extended version of Ink. Key subsystems:

| File | Purpose |
|------|---------|
| `ink/ink.tsx` | Core `Ink` class: manages the React reconciler, rendering pipeline, and input handling |
| `ink/root.ts` | `createRoot()` and `renderSync()` -- React-DOM-like root API. Defines `Root` and `Instance` types |
| `ink/reconciler.ts` | Custom React reconciler built on `react-reconciler`, using a Yoga-based DOM (`ink/dom.ts`) |
| `ink/dom.ts` | Virtual DOM nodes (`DOMElement`, `TextNode`) with Yoga layout attributes |
| `ink/renderer.ts` | Renders the virtual DOM tree to terminal output strings |
| `ink/render-node-to-output.ts` | Traverses DOM nodes, applies borders, computes positions |
| `ink/render-to-screen.ts` | Final screen buffer assembly |
| `ink/output.ts` | Output buffer abstraction |
| `ink/frame.ts` | Frame scheduling and flicker detection |
| `ink/screen.ts` | Screen state management |
| `ink/optimizer.ts` | Output diffing to minimize terminal writes |
| `ink/log-update.ts` | Manages terminal cursor for incremental redraws |
| `ink/focus.ts` | `FocusManager` for keyboard focus navigation |
| `ink/selection.ts` | Text selection support |
| `ink/terminal.ts` | Terminal capability detection (synchronized output, etc.) |
| `ink/terminal-querier.ts` | Queries terminal for capabilities via escape sequences |
| `ink/clearTerminal.ts` | Terminal clearing utilities |
| `ink/styles.ts` | Style system mapping to Yoga layout properties |
| `ink/stringWidth.ts` | Unicode-aware string width calculation |
| `ink/wrapAnsi.ts` | ANSI-aware text wrapping |
| `ink/bidi.ts` | Bidirectional text support |

#### Terminal I/O (`ink/termio/`)

Low-level terminal escape sequence handling:

| File | Purpose |
|------|---------|
| `ink/termio/dec.js` | DEC private mode sequences (cursor show/hide, alternate screen) |
| `ink/termio/osc.js` | Operating System Command sequences (tab status, hyperlinks) |

#### Ink Components (`ink/components/`)

Base UI primitives provided by the Ink layer:

| Component | File | Purpose |
|-----------|------|---------|
| `Box` | `ink/components/Box.tsx` | Flexbox container (maps to Yoga layout node) |
| `Text` | `ink/components/Text.tsx` | Text rendering with style support |
| `Button` | `ink/components/Button.tsx` | Clickable button with focus states |
| `Link` | `ink/components/Link.tsx` | Terminal hyperlinks |
| `Newline` | `ink/components/Newline.tsx` | Line break element |
| `Spacer` | `ink/components/Spacer.tsx` | Flexible space filler |
| `NoSelect` | `ink/components/NoSelect.tsx` | Content excluded from text selection |
| `RawAnsi` | `ink/components/RawAnsi.tsx` | Pass-through for raw ANSI sequences |
| `ScrollBox` | `ink/components/ScrollBox.tsx` | Scrollable container |
| `AlternateScreen` | `ink/components/AlternateScreen.tsx` | Terminal alternate screen buffer |
| `App` | `ink/components/App.tsx` | Ink-internal app context provider |
| `AppContext` | `ink/components/AppContext.ts` | Context for app-level state (exit callbacks) |
| `StdinContext` | `ink/components/StdinContext.ts` | Context for raw stdin access |
| `ClockContext` | `ink/components/ClockContext.tsx` | Clock/timer context for animations |
| `TerminalSizeContext` | `ink/components/TerminalSizeContext.tsx` | Reactive terminal dimensions |
| `TerminalFocusContext` | `ink/components/TerminalFocusContext.tsx` | Terminal focus/blur events |
| `CursorDeclarationContext` | `ink/components/CursorDeclarationContext.ts` | Cursor positioning declarations |

#### Ink Hooks (`ink/hooks/`)

| Hook | File | Purpose |
|------|------|---------|
| `useInput` | `use-input.ts` | Keyboard input handling with key parsing |
| `useApp` | `use-app.ts` | Access app context (exit function) |
| `useStdin` | `use-stdin.ts` | Raw stdin stream access |
| `useAnimationFrame` | `use-animation-frame.ts` | Request animation frame equivalent |
| `useInterval` / `useAnimationTimer` | `use-interval.ts` | Periodic callbacks |
| `useSelection` | `use-selection.ts` | Text selection state |
| `useTerminalViewport` | `use-terminal-viewport.ts` | Terminal dimensions (reactive) |
| `useTerminalFocus` | `use-terminal-focus.ts` | Terminal focus/blur state |
| `useTerminalTitle` | `use-terminal-title.ts` | Set terminal title bar text |
| `useTabStatus` | `use-tab-status.ts` | Terminal tab status text |
| `useDeclaredCursor` | `use-declared-cursor.ts` | Cursor position management |
| `useSearchHighlight` | `use-search-highlight.ts` | Search result highlighting |

### Theme Layer (`ink.ts`)

The `ink.ts` module wraps the raw Ink API to inject the design system. All `render()` and `createRoot()` calls go through this module, which wraps the rendered tree in a `ThemeProvider`:

```typescript
function withTheme(node: ReactNode): ReactNode {
  return createElement(ThemeProvider, null, node)
}

export async function createRoot(options?: RenderOptions): Promise<Root> {
  const root = await inkCreateRoot(options)
  return {
    ...root,
    render: node => root.render(withTheme(node)),
  }
}
```

Re-exports themed versions of Box and Text (`ThemedBox`, `ThemedText`) from the design system.

### Component Hierarchy

The application's React component tree follows this structure:

```
ThemeProvider                          (ink.ts - wraps all renders)
  FpsMetricsProvider                   (components/App.tsx)
    StatsProvider                      (components/App.tsx)
      AppStateProvider                 (components/App.tsx)
        REPL                           (screens/REPL.tsx)
          StatusLine                   (components/StatusLine.tsx)
          VirtualMessageList           (components/VirtualMessageList.tsx)
          TextInput / VimTextInput     (components/TextInput.tsx, VimTextInput.tsx)
          ToolUseLoader                (components/ToolUseLoader.tsx)
          Spinner                      (components/Spinner.tsx)
          ...dialogs and overlays
```

### App.tsx (`components/App.tsx`)

The top-level application wrapper provides three context layers:

```typescript
export function App({ getFpsMetrics, stats, initialState, children }: Props) {
  return (
    <FpsMetricsProvider getFpsMetrics={getFpsMetrics}>
      <StatsProvider store={stats}>
        <AppStateProvider initialState={initialState} onChangeAppState={onChangeAppState}>
          {children}
        </AppStateProvider>
      </StatsProvider>
    </FpsMetricsProvider>
  )
}
```

- **`FpsMetricsProvider`**: Exposes FPS tracking metrics for performance monitoring
- **`StatsProvider`**: Session statistics (token counts, costs, durations)
- **`AppStateProvider`**: Central application state with `onChangeAppState` callback for side-effects

### Screens (`screens/`)

| Screen | File | Purpose |
|--------|------|---------|
| `REPL` | `screens/REPL.tsx` | Main interactive session (input, messages, tool use) |
| `ResumeConversation` | `screens/ResumeConversation.tsx` | Session resume picker |
| `Doctor` | `screens/Doctor.tsx` | Installation diagnostics |

### Application Components (`components/`)

The `components/` directory contains ~144 files. Key categories:

- **Input**: `TextInput.tsx`, `VimTextInput.tsx`, `BaseTextInput.tsx`, `SearchBox.tsx`
- **Output**: `VirtualMessageList.tsx`, `StructuredDiff.tsx`, `CompactSummary.tsx`
- **Dialogs**: `TrustDialog/`, `AutoModeOptInDialog.tsx`, `BypassPermissionsModeDialog.tsx`, `CostThresholdDialog.tsx`, `BridgeDialog.tsx`, `WorkflowMultiselectDialog.tsx`
- **Status**: `StatusLine.tsx`, `StatusNotices.tsx`, `Spinner.tsx`, `ToolUseLoader.tsx`, `TokenWarning.tsx`
- **Navigation**: `TagTabs.tsx`, `ContextSuggestions.tsx`, `CustomSelect/`
- **Config/Setup**: `Onboarding.tsx`, `ThemePicker.tsx`, `Settings/`, `wizard/`
- **Design System**: `design-system/` (ThemedBox, ThemedText, ThemeProvider, color system)

---

## 4. Interactive Helpers (`interactiveHelpers.tsx`)

`interactiveHelpers.tsx` provides the bridge between the raw terminal I/O and the React rendering layer. It contains functions used during startup to display setup screens and manage the Ink root lifecycle.

### Key Functions

- **`showDialog<T>(root, renderer)`**: Generic dialog host. Wraps a renderer callback in a Promise, passing a `done` callback that resolves when the dialog completes. Used by all setup dialogs.

  ```typescript
  export function showDialog<T = void>(
    root: Root,
    renderer: (done: (result: T) => void) => React.ReactNode
  ): Promise<T>
  ```

- **`showSetupDialog<T>(root, renderer, options?)`**: Wraps `showDialog` with `AppStateProvider` and `KeybindingSetup` contexts. Every pre-REPL dialog uses this to get keybinding support and state management.

- **`exitWithError(root, message, beforeExit?)`**: Renders an error message through Ink (since `console.error` is swallowed by Ink's `patchConsole`), then unmounts and exits with code 1.

- **`exitWithMessage(root, message, options?)`**: General-purpose message display + exit. Supports custom color and exit code.

- **`renderAndRun(root, element)`**: Mounts the main UI element into the Ink root, starts deferred prefetches, waits for exit, then runs graceful shutdown.

  ```typescript
  export async function renderAndRun(root: Root, element: React.ReactNode): Promise<void> {
    root.render(element)
    startDeferredPrefetches()
    await root.waitUntilExit()
    await gracefulShutdown(0)
  }
  ```

- **`showSetupScreens(root, permissionMode, ...)`**: Orchestrates the pre-REPL setup sequence:
  1. Onboarding dialog (if first run or no theme set)
  2. Trust dialog (workspace trust boundary -- warns about untrusted repos, checks CLAUDE.md external includes)
  3. GrowthBook reinitialization after trust
  4. MCP server approval (`handleMcpjsonServerApprovals`)
  5. CLAUDE.md external includes approval
  6. Grove policy dialog (if eligible)
  7. Environment variable application (post-trust)
  8. Telemetry initialization
  9. Dangerous-mode permission dialog (if applicable)
  10. Auto-mode opt-in dialog (if applicable)

- **`completeOnboarding()`**: Saves onboarding completion state to global config.

- **`getRenderContext()`**: Creates the Ink root with proper render options (referenced in `main.tsx` but defined via `getBaseRenderOptions()`).

### Multi-line Input and Text Input

The `TextInput.tsx` and `VimTextInput.tsx` components handle complex text input scenarios:

- **Multi-line input**: Supported via Option+Enter (macOS) or configured keybinding. The `terminalSetup` command configures terminal-specific key bindings.
- **Cursor positioning**: Managed through `CursorDeclarationContext` and `useDeclaredCursor` hook.
- **Vim mode**: Full vim-style editing via `VimTextInput.tsx`, toggled with `/vim`.
- **Early input capture**: `utils/earlyInput.ts` (`seedEarlyInput`, `stopCapturingEarlyInput`) captures keystrokes during startup before the REPL mounts.

---

## 5. Dialog System (`dialogLaunchers.tsx`)

`dialogLaunchers.tsx` provides thin launcher functions for one-off dialog components used in `main.tsx`. Each launcher dynamically imports its component and wires the `done` callback. This pattern keeps `main.tsx` free of JSX and defers heavy component imports until needed.

### Dialog Launchers

| Function | Component | Purpose |
|----------|-----------|---------|
| `launchSnapshotUpdateDialog()` | `SnapshotUpdateDialog` | Agent memory snapshot update prompt. Returns `'merge'`, `'keep'`, or `'replace'` |
| `launchInvalidSettingsDialog()` | `InvalidSettingsDialog` | Displays settings validation errors with continue/exit options |
| `launchAssistantSessionChooser()` | `AssistantSessionChooser` | Pick a bridge/assistant session to attach to |
| `launchAssistantInstallWizard()` | `NewInstallWizard` | Install wizard when `claude assistant` finds zero sessions |
| `launchResumeChooser()` | `ResumeConversation` | Interactive session resume picker |
| `launchTeleportResumeWrapper()` | `TeleportResumeWrapper` | Resume a teleported (remote) session |
| `launchTeleportRepoMismatchDialog()` | `TeleportRepoMismatchDialog` | Warns when teleported session's repo doesn't match local |

### Additional Setup Dialogs (in `interactiveHelpers.tsx` flow)

These are rendered inline during `showSetupScreens()`:

| Dialog | Purpose |
|--------|---------|
| `Onboarding` | First-run theme picker and welcome |
| `TrustDialog` | Workspace trust boundary (warns about untrusted repos) |
| `ClaudeMdExternalIncludesDialog` | Approval for CLAUDE.md external file includes |
| `GroveDialog` | Policy acceptance dialog |
| `BypassPermissionsModeDialog` | Dangerous permissions mode confirmation |
| `AutoModeOptInDialog` | Auto-mode opt-in prompt |
| `BridgeDialog` | Bridge/remote-control connection setup |
| `CostThresholdDialog` | Spending threshold warning |
| `ChannelDowngradeDialog` | Channel server downgrade confirmation |

### Pattern

All dialogs follow the same pattern:

```typescript
export async function launchSomeDialog(root: Root, props: {...}): Promise<ResultType> {
  const { SomeComponent } = await import('./components/SomeComponent.js')
  return showSetupDialog<ResultType>(root, done => (
    <SomeComponent {...props} onComplete={done} onCancel={() => done(defaultValue)} />
  ))
}
```

---

## 6. Slash Command System (`commands/` and `commands.ts`)

Slash commands are locally-evaluated commands prefixed with `/` that are not sent to the LLM. They are registered in `commands.ts` and their implementations live in the `commands/` directory.

### Command Types (`types/command.ts`)

Commands have three type variants:

| Type | Description |
|------|-------------|
| `prompt` | Generates content blocks sent to the model (e.g., `/review`, `/pr_comments`) |
| `local` | Runs locally, returns text result (e.g., `/cost`, `/stats`) |
| `local-jsx` | Runs locally, renders a React component in the REPL (e.g., `/config`, `/resume`) |

Each command also has:
- **`name`**: The slash command name (e.g., `'clear'`)
- **`description`**: User-visible help text
- **`availability`**: Auth/provider requirements (`'all'`, `'1p'`, `'claude-ai'`, etc.)
- **`isEnabled()`**: Runtime gate (GrowthBook flags, platform, env vars)
- **`source`**: Origin (`'builtin'`, `'mcp'`, `'plugin'`, `'bundled'`)

### Command Resolution (`commands.ts`)

- **`getCommands(cwd)`**: Returns the full command list, including built-in commands, skill-dir commands, bundled skills, plugin commands, and MCP skill commands.
- **`findCommand(name, commands)`**: Looks up a command by name.
- **`filterCommandsForRemoteMode(commands)`**: Filters to `REMOTE_SAFE_COMMANDS` for remote sessions.
- **`isBridgeSafeCommand(cmd)`**: Checks if a command is safe for bridge mode.
- **`meetsAvailabilityRequirement(cmd)`**: Checks auth/provider prerequisites.

### Complete Command Reference

#### Session Management

| Command | Description |
|---------|-------------|
| `/clear` | Clear conversation history and free up context |
| `/compact` | Clear history but keep a summary in context. Accepts optional summarization instructions |
| `/resume` | Resume a previous conversation (interactive picker or by session ID) |
| `/session` | Show remote session URL and QR code |
| `/rename` | Rename the current conversation |
| `/branch` | Create a branch of the current conversation at this point |
| `/tag` | Toggle a searchable tag on the current session |
| `/export` | Export the current conversation to a file or clipboard |
| `/copy` | Copy Claude's last response to clipboard (`/copy N` for Nth-latest) |
| `/exit` | Exit the REPL |

#### Code & Git

| Command | Description |
|---------|-------------|
| `/diff` | View uncommitted changes and per-turn diffs |
| `/review` | Trigger a code review of pending changes |
| `/ultrareview` | Enhanced code review (extended variant) |
| `/security-review` | Security-focused review of pending changes on the current branch |
| `/pr_comments` | Get comments from a GitHub pull request |
| `/rewind` | Restore code and/or conversation to a previous point |
| `/plan` | Enable plan mode or view the current session plan |

#### Configuration & Settings

| Command | Description |
|---------|-------------|
| `/config` | Open config panel |
| `/theme` | Change the terminal color theme |
| `/color` | Set the prompt bar color for this session |
| `/model` | Switch the model for the current session |
| `/effort` | Set effort level for model usage (low/medium/high/max) |
| `/fast` | Toggle fast mode (dynamic description based on current state) |
| `/permissions` | Manage allow and deny tool permission rules |
| `/hooks` | View hook configurations for tool events |
| `/keybindings` | Open or create your keybindings configuration file |
| `/privacy-settings` | View and update privacy settings |
| `/output-style` | Deprecated: use `/config` to change output style |
| `/memory` | Edit Claude memory files (CLAUDE.md) |
| `/vim` | Toggle between Vim and Normal editing modes |
| `/sandbox-toggle` | Toggle sandbox mode |
| `/advisor` | Configure the advisor model |

#### Context & Diagnostics

| Command | Description |
|---------|-------------|
| `/context` | Visualize current context usage as a colored grid |
| `/files` | List all files currently in context |
| `/cost` | Show total cost and duration of the current session |
| `/usage` | Show plan usage limits |
| `/stats` | Show Claude Code usage statistics and activity |
| `/status` | Show version, model, account, API connectivity, and tool statuses |
| `/doctor` | Diagnose and verify your Claude Code installation and settings |

#### Agent & Tools

| Command | Description |
|---------|-------------|
| `/agents` | Manage agent configurations |
| `/skills` | List available skills |
| `/tasks` | List and manage background tasks |
| `/mcp` | Manage MCP servers (add, remove, list, configure) |
| `/plugin` | Manage Claude Code plugins |
| `/reload-plugins` | Activate pending plugin changes in the current session |

#### Account & Auth

| Command | Description |
|---------|-------------|
| `/login` | Sign in with your Anthropic account (or switch accounts) |
| `/logout` | Sign out from your Anthropic account |
| `/passes` | Manage passes (dynamic description) |
| `/extra-usage` | Configure extra usage to keep working when limits are hit |
| `/rate-limit-options` | Show options when rate limit is reached |
| `/upgrade` | Upgrade to Max for higher rate limits and more Opus |

#### IDE & Integration

| Command | Description |
|---------|-------------|
| `/ide` | Manage IDE integrations and show status |
| `/desktop` | Continue the current session in Claude Desktop |
| `/chrome` | Claude in Chrome settings |
| `/mobile` | Show QR code to download the Claude mobile app |
| `/install-github-app` | Set up Claude GitHub Actions for a repository |
| `/install-slack-app` | Install the Claude Slack app |
| `/terminal-setup` | Enable Option+Enter key binding and other terminal settings |
| `/stickers` | Order Claude Code stickers |

#### Information & Help

| Command | Description |
|---------|-------------|
| `/help` | Show help and available commands |
| `/release-notes` | View release notes |
| `/feedback` | Submit feedback about Claude Code |
| `/add-dir` | Add a new working directory |
| `/remote-env` | Configure the default remote environment for teleport sessions |
| `/btw` | Ask a quick side question without interrupting the main conversation |

#### Display & UI

| Command | Description |
|---------|-------------|
| `/statusline` | Set up Claude Code's status line UI |
| `/thinkback` | Your 2025 Claude Code Year in Review |
| `/thinkback-play` | Play the thinkback animation |
| `/insights` | Generate a report analyzing your Claude Code sessions (lazy-loaded, ~113KB) |

#### Feature-Gated Commands

These commands are conditionally registered based on build-time feature flags:

| Command | Feature Flag | Description |
|---------|-------------|-------------|
| `/proactive` | `PROACTIVE` or `KAIROS` | Start proactive autonomous mode |
| `/brief` | `KAIROS` or `KAIROS_BRIEF` | Enable SendUserMessage tool |
| `/assistant` | `KAIROS` | Assistant mode management |
| `/bridge` | `BRIDGE_MODE` | Connect terminal for remote-control sessions |
| `/remote-control-server` | `DAEMON` + `BRIDGE_MODE` | Remote control server management |
| `/voice` | `VOICE_MODE` | Toggle voice mode |
| `/force-snip` | `HISTORY_SNIP` | Force history snipping |
| `/workflows` | `WORKFLOW_SCRIPTS` | Workflow management |
| `/remote-setup` | `CCR_REMOTE_SETUP` | Setup Claude Code on the web |
| `/subscribe-pr` | `KAIROS_GITHUB_WEBHOOKS` | Subscribe to PR webhooks |
| `/ultraplan` | `ULTRAPLAN` | Ultra-planning mode |
| `/torch` | `TORCH` | Torch integration |
| `/peers` | `UDS_INBOX` | Peer session communication |
| `/fork` | `FORK_SUBAGENT` | Fork as sub-agent |
| `/buddy` | `BUDDY` | Buddy mode |

#### Internal-Only Commands (Ant)

Available only when `USER_TYPE === 'ant'`:

`/backfill-sessions`, `/break-cache`, `/bughunter`, `/commit`, `/commit-push-pr`, `/ctx_viz`, `/good-claude`, `/issue`, `/init-verifiers`, `/mock-limits`, `/bridge-kick`, `/version`, `/reset-limits`, `/onboarding`, `/share`, `/summary`, `/teleport`, `/ant-trace`, `/perf-issue`, `/env`, `/oauth-refresh`, `/debug-tool-call`, `/agents-platform`, `/autofix-pr`

#### Dynamic Commands

Beyond built-in commands, the system also loads:
- **Skill directory commands**: From `skills/loadSkillsDir.ts` via `getSkillDirCommands()`
- **Bundled skills**: From `skills/bundledSkills.ts` via `getBundledSkills()`
- **Plugin commands**: From `utils/plugins/loadPluginCommands.ts` via `getPluginCommands()`
- **Plugin skill commands**: Via `getBuiltinPluginSkillCommands()`
- **MCP skill commands**: Via `getMcpSkillCommands()`

---

## 7. Output System (`cli/`)

### Headless Output (`cli/print.ts`)

`cli/print.ts` contains `runHeadless()`, the main entry point for non-interactive (`-p`) mode. It handles:

- Tool assembly and filtering (`assembleToolPool`, `filterToolsByDenyRules`)
- Message queue management (`enqueue`, `dequeue`, `subscribeToCommandQueue`)
- Session state notifications (`notifySessionStateChanged`, `notifySessionMetadataChanged`)
- MCP server reconciliation (`reconcileMcpServers`, `handleMcpSetServers`)
- Conversation recovery (`loadConversationForResume`)
- Permission prompt forwarding (via `StructuredIO` or `permissionPromptTool`)

Key exported functions:
- **`runHeadless()`** (line ~455): Main headless execution loop
- **`createCanUseToolWithPermissionPrompt()`** (line ~4149): Builds the permission decision function for SDK mode
- **`getCanUseToolFn()`** (line ~4267): Resolves the appropriate tool permission function
- **`removeInterruptedMessage()`** (line ~4875): Cleans up interrupted assistant messages
- **`handleOrphanedPermissionResponse()`** (line ~5241): Handles late/duplicate permission responses
- **`joinPromptValues()`** (line ~428): Merges multi-part prompt values
- **`canBatchWith()`** (line ~443): Determines if messages can be batched together

### Structured I/O (`cli/structuredIO.ts`)

The `StructuredIO` class provides the SDK protocol layer for structured JSON communication over stdin/stdout:

```typescript
export class StructuredIO {
  readonly structuredInput: AsyncGenerator<StdinMessage | SDKMessage>
  readonly outbound: Stream<StdoutMessage>

  constructor(input: AsyncIterable<string>, replayUserMessages?: boolean)
}
```

It handles:
- **SDK control requests/responses**: Permission prompts (`can_use_tool`), elicitation, hook callbacks
- **Message serialization**: NDJSON-safe output via `ndjsonSafeStringify()`
- **Duplicate detection**: Tracks resolved `tool_use_id`s to prevent duplicate permission processing
- **Session state restoration**: `restoredWorkerState` for CCR worker restart

The companion `RemoteIO` class (`cli/remoteIO.ts`) extends this for remote/WebSocket transport.

### Output Formats

| Format | Flag | Transport |
|--------|------|-----------|
| Plain text | `--output-format text` (default) | Direct stdout |
| Single JSON | `--output-format json` | Buffered JSON object |
| Streaming JSON | `--output-format stream-json` | NDJSON lines via `StructuredIO.outbound` |

### SDK Control Protocol

When using `stream-json` format, the SDK protocol supports bidirectional control messages:

| Direction | Message Type | Purpose |
|-----------|-------------|---------|
| Outbound | `stream_event` | Token chunks, tool use, status updates |
| Outbound | `control_request` | Permission prompts, elicitation |
| Inbound | `control_response` | Permission decisions, elicitation results |
| Inbound | `user_message` | New user messages |
| Inbound | `set_servers` | Dynamic MCP server configuration |
| Inbound | `resume` | Resume a session |

---

## 8. Key Source Files Reference

| File | Purpose |
|------|---------|
| `entrypoints/cli.tsx` | Process entrypoint: fast-path dispatch, bridge, daemon, background sessions |
| `entrypoints/init.ts` | Core initialization: configs, TLS, env vars, telemetry, graceful shutdown |
| `entrypoints/mcp.ts` | MCP server mode: exposes tools via Model Context Protocol |
| `entrypoints/sdk/` | SDK type definitions and control message schemas |
| `main.tsx` | Main loop: Commander parsing, setup screens, REPL launch (~5000+ lines) |
| `commands.ts` | Command registry: imports, categorizes, and exports all slash commands |
| `commands/` | Individual command implementations (~100+ directories and files) |
| `types/command.ts` | Command type definitions: `PromptCommand`, `LocalCommand`, `LocalJSXCommand` |
| `ink.ts` | Theme-wrapped Ink API: `render()`, `createRoot()`, component re-exports |
| `ink/root.ts` | Ink root management: `Root`, `Instance`, `createRoot()`, `renderSync()` |
| `ink/ink.tsx` | Core Ink engine: reconciler integration, rendering pipeline |
| `ink/reconciler.ts` | Custom React reconciler using `react-reconciler` + Yoga layout |
| `ink/dom.ts` | Virtual DOM for terminal: `DOMElement`, `TextNode`, layout attributes |
| `ink/frame.ts` | Frame scheduling and flicker detection |
| `ink/renderer.ts` | DOM-to-string rendering |
| `ink/components/` | Base Ink components: Box, Text, Button, ScrollBox, etc. |
| `ink/hooks/` | Ink hooks: useInput, useApp, useStdin, useTerminalViewport, etc. |
| `ink/termio/` | Low-level terminal escape sequences (DEC, OSC) |
| `components/App.tsx` | Top-level app wrapper: FPS, Stats, and AppState providers |
| `components/` | ~144 application components: dialogs, inputs, status, settings |
| `screens/REPL.tsx` | Main interactive REPL screen |
| `screens/ResumeConversation.tsx` | Session resume picker screen |
| `screens/Doctor.tsx` | Installation diagnostics screen |
| `replLauncher.tsx` | `launchRepl()`: mounts `App` + `REPL` into Ink root |
| `interactiveHelpers.tsx` | Setup helpers: `showDialog`, `showSetupScreens`, `renderAndRun`, `exitWithError` |
| `dialogLaunchers.tsx` | Thin launchers for pre-REPL dialogs (snapshot, resume, settings, teleport) |
| `cli/print.ts` | Headless mode: `runHeadless()`, permission handling, MCP reconciliation |
| `cli/structuredIO.ts` | `StructuredIO` class: SDK JSON protocol over stdin/stdout |
| `cli/remoteIO.ts` | `RemoteIO`: WebSocket transport extension of StructuredIO |
| `state/AppStateStore.ts` | `AppState` type and `getDefaultAppState()` |
| `state/AppState.tsx` | `AppStateProvider` React context |
| `state/onChangeAppState.ts` | Side-effect handler for state transitions |
| `context/stats.ts` | `StatsStore` and `StatsProvider` for session metrics |
| `context/fpsMetrics.ts` | `FpsMetricsProvider` for rendering performance |
| `keybindings/KeybindingProviderSetup.ts` | Keybinding context for dialogs |
| `utils/earlyInput.ts` | Pre-REPL keystroke capture |
| `utils/renderOptions.ts` | `getBaseRenderOptions()` for Ink configuration |
| `utils/fpsTracker.ts` | FPS measurement for terminal rendering |
| `skills/loadSkillsDir.ts` | Dynamic skill directory loading |
| `skills/bundledSkills.ts` | Built-in bundled skills |
| `utils/plugins/loadPluginCommands.ts` | Plugin command loading |
