# Plugin System and Skills System

## 1. Overview -- Extensibility Architecture

Claude Code exposes two complementary extension mechanisms:

**Plugins** are packages of capabilities (skills, hooks, MCP servers, LSP servers, agents, output styles) that are discovered from marketplaces or local directories, installed into a versioned cache, and managed through settings at multiple scopes. A single plugin can bundle several of these components together under one manifest.

**Skills** are individual prompt-based commands that inject specialized instructions and domain knowledge into the conversation. They are the primary unit of model-facing extensibility. Skills can originate from bundled code, user-authored markdown files, plugins, or MCP servers.

The two systems compose: plugins deliver skills (among other things), while skills can also exist independently of the plugin system.

### Component Hierarchy

```
Plugin (plugin.json manifest)
  |-- skills/         -> Skill markdown files
  |-- commands/       -> Legacy command markdown (deprecated alias for skills)
  |-- agents/         -> Agent definition files
  |-- hooks.json      -> Hook configuration
  |-- output-styles/  -> Output formatting
  |-- MCP servers     -> Declared in manifest
  |-- LSP servers     -> Declared in manifest
```

---

## 2. Plugin System

### 2.1 Plugin Types and Definitions

Defined in `types/plugin.ts` and `utils/plugins/schemas.ts`.

#### BuiltinPluginDefinition

Plugins that ship compiled into the CLI binary. They appear in the `/plugin` UI and can be toggled by users. Use the `{name}@builtin` identifier format.

```typescript
type BuiltinPluginDefinition = {
  name: string
  description: string
  version?: string
  skills?: BundledSkillDefinition[]
  hooks?: HooksSettings
  mcpServers?: Record<string, McpServerConfig>
  isAvailable?: () => boolean    // hide when system lacks capabilities
  defaultEnabled?: boolean       // defaults to true
}
```

#### LoadedPlugin

The runtime representation of any resolved plugin, regardless of source:

```typescript
type LoadedPlugin = {
  name: string
  manifest: PluginManifest
  path: string
  source: string                 // e.g. "my-plugin@my-marketplace"
  repository: string
  enabled?: boolean
  isBuiltin?: boolean
  sha?: string                   // Git commit SHA for version pinning
  commandsPath?: string
  commandsPaths?: string[]
  skillsPath?: string
  skillsPaths?: string[]
  agentsPath?: string
  agentsPaths?: string[]
  outputStylesPath?: string
  outputStylesPaths?: string[]
  hooksConfig?: HooksSettings
  mcpServers?: Record<string, McpServerConfig>
  lspServers?: Record<string, LspServerConfig>
  settings?: Record<string, unknown>
}
```

#### PluginComponent

The five component types a plugin can contribute:

```typescript
type PluginComponent = 'commands' | 'agents' | 'skills' | 'hooks' | 'output-styles'
```

#### PluginError

A discriminated union of 20+ typed error variants covering every failure mode from git auth failures to marketplace policy blocks. Each variant carries structured context (paths, URLs, server names) for debugging. Key variants include:

| Error Type | Description |
|---|---|
| `generic-error` | Catch-all with free-form message |
| `plugin-not-found` | Plugin ID not found in marketplace |
| `path-not-found` | Component path missing on disk |
| `git-auth-failed` | SSH/HTTPS authentication failure |
| `manifest-parse-error` | plugin.json could not be parsed |
| `manifest-validation-error` | plugin.json failed schema validation |
| `marketplace-blocked-by-policy` | Enterprise policy blocked the marketplace |
| `mcp-server-suppressed-duplicate` | MCP server skipped because of duplicate |
| `dependency-unsatisfied` | Required dependency not found or disabled |
| `plugin-cache-miss` | Plugin not in local cache |

The `getPluginErrorMessage()` function converts any `PluginError` to a human-readable string.

#### PluginLoadResult

```typescript
type PluginLoadResult = {
  enabled: LoadedPlugin[]
  disabled: LoadedPlugin[]
  errors: PluginError[]
}
```

#### Installation Scopes

Plugins can be installed at three user-facing scopes, plus a managed scope for enterprise:

| Scope | Settings Location | Description |
|---|---|---|
| `user` | `~/.claude/settings.json` | Available in all projects |
| `project` | `.claude/settings.json` (repo root) | Shared via version control |
| `local` | `.claude/settings.local.json` | Project-local, not committed |
| `managed` | Managed settings path | Enterprise-administered |

### 2.2 Plugin Discovery and Loading

Implemented in `utils/plugins/pluginLoader.ts`.

#### Discovery Sources (in precedence order)

1. **Marketplace-based plugins** -- Identified by `plugin@marketplace` format in settings. The marketplace is a git repository containing a `marketplace.json` that catalogs available plugins. Each entry specifies the plugin's source (git repo, NPM package, etc.), version, and metadata.

2. **Session-only plugins** -- From `--plugin-dir` CLI flag or SDK `plugins` option. These are inline directory-based plugins that exist only for the session.

3. **Built-in plugins** -- Registered programmatically at startup via `registerBuiltinPlugin()` in `plugins/builtinPlugins.ts`. Identified by the `@builtin` marketplace sentinel.

#### Plugin Directory Structure

```
my-plugin/
+-- plugin.json          # Manifest (name, description, version, component paths)
+-- skills/              # Skill markdown files (SKILL.md or *.md)
+-- commands/            # Legacy command directory (deprecated)
+-- agents/              # Agent definition markdown files
+-- hooks.json           # Hook definitions
+-- output-styles/       # Output style configurations
```

#### Loading Pipeline

The `loadAllPlugins()` function (memoized) orchestrates loading:

1. Read `enabledPlugins` from all settings scopes (user, project, local, managed).
2. For each plugin ID, resolve through marketplace cache or inline directory.
3. Validate the manifest against `PluginManifestSchema`.
4. Check enterprise policy (`isPluginBlockedByPolicy`, `isSourceAllowedByPolicy`).
5. Resolve dependencies via `dependencyResolver.ts`.
6. Return `PluginLoadResult` with enabled, disabled, and error arrays.

#### Versioned Caching

Plugins are cached in a versioned directory structure under `~/.claude/plugins/cache/`:

```
~/.claude/plugins/cache/
+-- <marketplace>/
    +-- <plugin>/
        +-- <version>/
            +-- plugin.json
            +-- skills/
            +-- ...
```

The `CLAUDE_CODE_PLUGIN_CACHE_DIR` environment variable overrides the base cache directory. Zip-based caching (`zipCache.ts`) is also supported for space efficiency.

#### Marketplace Management

Marketplaces are git repositories that contain a `marketplace.json` file listing available plugins. Key files:

- `utils/plugins/marketplaceManager.ts` -- Loading and caching marketplace data
- `utils/plugins/reconciler.ts` -- Synchronizes declared marketplaces with local clones
- `utils/plugins/officialMarketplace.ts` / `officialMarketplaceGcs.ts` -- Official Anthropic marketplace handling

Official marketplace names (e.g., `claude-code-marketplace`, `anthropic-plugins`) are reserved and validated against the `anthropics` GitHub organization in `utils/plugins/schemas.ts`.

### 2.3 Plugin Lifecycle

#### Installation

Core operations live in `services/plugins/pluginOperations.ts` as pure library functions (no process.exit, no console output). CLI wrappers in `services/plugins/pluginCliCommands.ts` add output formatting and process management.

**`installPluginOp(plugin, scope)`**

1. Parse the plugin identifier (name and optional marketplace).
2. Look up the plugin in the marketplace.
3. Resolve and download the plugin source to the versioned cache.
4. Write the plugin ID to `enabledPlugins` in the appropriate scope's settings file.

**`uninstallPluginOp(plugin, scope, deleteData)`**

1. Find the plugin in settings across all scopes.
2. Remove from `enabledPlugins`.
3. Optionally delete cached data and plugin-specific settings via `deletePluginDataDir` and `deletePluginOptions`.
4. Check for reverse dependents and warn.

**`enablePluginOp` / `disablePluginOp`**

Toggle the boolean value in `enabledPlugins` at the given scope. Built-in plugins use `{name}@builtin` as their ID.

**`updatePluginOp(plugin, scope)`**

Re-fetch the plugin from its marketplace source, compute a new version, and copy to a new versioned cache slot.

#### Background Installation

`services/plugins/PluginInstallationManager.ts` handles non-blocking startup installation:

1. Compute the diff between declared marketplaces (from settings) and materialized ones (cloned on disk).
2. For missing or source-changed marketplaces, run `reconcileMarketplaces()` with progress callbacks mapped to `AppState.plugins.installationStatus`.
3. On new installs, auto-refresh plugins via `refreshActivePlugins()`.
4. On updates, set `needsRefresh` flag and notify user to run `/reload-plugins`.

#### Hot Reload

After initial load, plugins do not auto-refresh. The `useManagePlugins` hook in `hooks/useManagePlugins.ts` detects `needsRefresh` and shows a notification. The user explicitly runs `/reload-plugins` which calls `refreshActivePlugins()` to:

- Clear all plugin caches (loader, commands, hooks, MCP, LSP).
- Re-run `loadAllPlugins()`.
- Bump `pluginReconnectKey` to re-establish MCP connections.
- Reinitialize LSP server manager.

### 2.4 Plugin Management

#### useManagePlugins Hook

`hooks/useManagePlugins.ts` -- React hook that drives initial plugin load on REPL mount:

1. Calls `loadAllPlugins()` to get enabled/disabled/errors.
2. Runs `detectAndUninstallDelistedPlugins()` for blocklist enforcement.
3. Surfaces flagged-plugin notifications.
4. Loads plugin commands (`getPluginCommands`), agents (`loadPluginAgents`), hooks (`loadPluginHooks`), MCP servers, and LSP servers.
5. Merges everything into `AppState.plugins`.
6. Emits `tengu_plugins_loaded` telemetry with counts of each component type.

#### CLI Commands

The `/plugin` slash command (in `commands/plugin/`) renders the `PluginSettings` component, which provides an interactive TUI for:

- **ManagePlugins** -- Enable/disable installed plugins
- **BrowseMarketplace** -- Discover and install plugins from configured marketplaces
- **ManageMarketplaces** -- Add/remove marketplace sources
- **AddMarketplace** -- Register a new marketplace by URL
- **ValidatePlugin** -- Check plugin manifest and structure
- **PluginTrustWarning** -- Security confirmation before enabling untrusted plugins

CLI (non-interactive) commands in `services/plugins/pluginCliCommands.ts`:

| Command | Function |
|---|---|
| `claude plugin install <id>` | `installPlugin(plugin, scope)` |
| `claude plugin uninstall <id>` | `uninstallPlugin(plugin, scope)` |
| `claude plugin enable <id>` | `enablePlugin(plugin, scope)` |
| `claude plugin disable <id>` | `disablePlugin(plugin, scope)` |
| `claude plugin disable-all` | `disableAllPlugins()` |
| `claude plugin update <id>` | `updatePluginCli(plugin, scope)` |

All CLI commands emit telemetry (`tengu_plugin_*_cli`) and use the structured error handler `handlePluginCommandError`.

#### Plugin Components Loading

Each component type has a dedicated loader in `utils/plugins/`:

| Component | Loader | Description |
|---|---|---|
| Commands/Skills | `loadPluginCommands.ts` | Walks `skills/` and `commands/` dirs, parses markdown frontmatter |
| Agents | `loadPluginAgents.ts` | Loads agent definition markdown files |
| Hooks | `loadPluginHooks.ts` | Parses hooks.json, converts to native `PluginHookMatcher` format |
| MCP Servers | `mcpPluginIntegration.ts` | Extracts MCP server configs from manifest, deduplicates |
| LSP Servers | `lspPluginIntegration.ts` | Extracts LSP server configs from manifest |
| Output Styles | `loadPluginOutputStyles.ts` | Loads custom output formatting |

---

## 3. Skills System

### 3.1 Skill Architecture

A skill is a prompt-based command represented by the `Command` type (specifically `PromptCommand`). Skills inject specialized instructions into the conversation when invoked, either by the user typing a slash command or by the model calling the `Skill` tool.

Skills consist of:

- **Name** -- The slash-command identifier (e.g., `commit`, `review-pr`, `pdf`).
- **Description / whenToUse** -- Text shown in the skill listing for model discovery.
- **Prompt content** -- The markdown instructions loaded on invocation (lazy-loaded, not at startup).
- **Metadata** -- Allowed tools, model override, effort level, execution context (inline vs. fork), hooks, argument patterns.

#### Execution Contexts

Skills run in one of two contexts:

| Context | Description |
|---|---|
| `inline` (default) | Skill prompt is injected into the current conversation as user messages. The model continues in the same context with the skill's instructions. |
| `fork` | Skill runs in an isolated sub-agent via `runAgent()`. The sub-agent has its own token budget and message history. Results are extracted and returned to the parent conversation. |

#### Skill Sources

Skills can come from multiple sources, tracked by `Command.source` and `Command.loadedFrom`:

| Source | loadedFrom | Description |
|---|---|---|
| `bundled` | `bundled` | Compiled into the CLI binary, registered at startup |
| `userSettings` | `skills` | `~/.claude/skills/*.md` files |
| `projectSettings` | `skills` | `.claude/skills/*.md` files |
| `policySettings` | `skills` | Managed settings path skills |
| `plugin` | `plugin` | Skills from installed plugins |
| `mcp` | `mcp` | Skills exposed by MCP servers |

### 3.2 Skill Registration and Discovery

#### Bundled Skills

Defined in `skills/bundled/`. Each skill file exports a `register*Skill()` function that calls `registerBundledSkill()` from `skills/bundledSkills.ts`. All bundled skills are initialized in `skills/bundled/index.ts::initBundledSkills()`.

Currently registered bundled skills include:

| Skill | Feature Gate | Description |
|---|---|---|
| `update-config` | always | Configure Claude Code settings |
| `keybindings` | always | Customize keyboard shortcuts |
| `verify` | ant-only | Verify code changes work correctly |
| `debug` | always | Systematic debugging |
| `lorem-ipsum` | always | Generate placeholder text |
| `skillify` | always | Create new skills |
| `remember` | always | Save learnings to CLAUDE.md |
| `simplify` | always | Review code for quality |
| `batch` | always | Batch operations |
| `stuck` | always | Help when stuck on a problem |
| `loop` | AGENT_TRIGGERS | Run commands on recurring intervals |
| `schedule` | AGENT_TRIGGERS_REMOTE | Schedule remote agent triggers |
| `claude-api` | BUILDING_CLAUDE_APPS | Build apps with Claude API/SDKs |
| `claude-in-chrome` | auto-detect | Chrome browser integration |

The `BundledSkillDefinition` type:

```typescript
type BundledSkillDefinition = {
  name: string
  description: string
  aliases?: string[]
  whenToUse?: string
  allowedTools?: string[]
  model?: string
  disableModelInvocation?: boolean
  userInvocable?: boolean
  isEnabled?: () => boolean
  hooks?: HooksSettings
  context?: 'inline' | 'fork'
  agent?: string
  files?: Record<string, string>  // Reference files extracted to disk on first use
  getPromptForCommand: (args, context) => Promise<ContentBlockParam[]>
}
```

When a bundled skill has `files`, they are lazily extracted to a deterministic directory under `getBundledSkillsRoot()` on first invocation. The skill prompt is prefixed with the base directory path so the model can Read/Grep reference files.

#### File-Based Skills

Loaded by `skills/loadSkillsDir.ts`. Skill files are markdown documents with YAML frontmatter:

```markdown
---
description: What this skill does
allowed-tools: Edit, Read, Bash
when-to-use: Use when the user asks to...
model: opus
context: fork
argument-hint: <file-path>
hooks:
  PreToolUse:
    - matcher: Bash
      hooks:
        - type: command
          command: echo "pre-bash hook"
---

Your skill instructions here. Use $ARGUMENTS for user-provided args.
```

Skills are discovered from these directories (in order):

1. `~/.claude/skills/` (user scope)
2. `.claude/skills/` (project scope)
3. Managed settings path skills (policy scope)
4. Plugin `skills/` directories

Within each directory, skill files can be:
- `SKILL.md` inside a named subdirectory (e.g., `skills/my-skill/SKILL.md`)
- `*.md` files directly in the skills directory (name derived from filename)

The `parseSkillFrontmatterFields()` function extracts all metadata from frontmatter including: description, allowed-tools, when-to-use, model, effort, context (fork/inline), argument-hint, hooks, paths, shell, disable-model-invocation, user-invocable, and version.

#### MCP Skills

MCP servers can expose skills through the `mcpSkillBuilders.ts` registry. The `registerMCPSkillBuilders()` function connects the MCP subsystem to the skill loading pipeline, allowing MCP servers to provide skill commands that appear in the skill listing alongside local and bundled skills.

#### Skill Change Detection

`utils/skills/skillChangeDetector.ts` uses `chokidar` file watchers on skill directories. When skill files change on disk (e.g., during plugin auto-update), it debounces rapid changes, clears skill caches, and triggers a reload. The debounce window (`FILE_STABILITY_THRESHOLD_MS = 1000ms`) prevents cascading reloads when many files change simultaneously.

### 3.3 Skill Execution (SkillTool)

The `Skill` tool (`tools/SkillTool/`) is the model-facing interface for invoking skills. It is defined as a standard Claude Code tool with input validation, permission checking, and result rendering.

#### Tool Definition

```typescript
// tools/SkillTool/constants.ts
const SKILL_TOOL_NAME = 'Skill'

// Input schema
z.object({
  skill: z.string().describe('The skill name'),
  args: z.string().optional().describe('Optional arguments'),
})
```

#### Execution Flow

1. **Validation** (`validateInput`):
   - Normalize the skill name (strip leading `/`).
   - Look up the command in local commands + MCP skills.
   - Reject if command not found, has `disableModelInvocation`, or is not `type: 'prompt'`.

2. **Permission Check** (`checkPermissions`):
   - Check deny rules, then allow rules (using exact match and `prefix:*` patterns).
   - Auto-allow skills with only safe properties (no hooks, allowed-tools override, etc.).
   - For skills requiring permission, prompt the user with suggestions to create allow rules.

3. **Execution** (`call`):
   - For `context: 'fork'` skills: execute via `executeForkedSkill()` which runs a sub-agent with `runAgent()`, collects messages, and returns the extracted result text.
   - For inline skills: call `processPromptSlashCommand()` to expand the skill content, apply `$ARGUMENTS` substitution, and return the skill prompt as new user messages injected into the conversation.
   - Apply context modifications: allowed-tools overrides, model overrides, effort overrides.

4. **Result Mapping** (`mapToolResultToToolResultBlockParam`):
   - Forked skills: return the full result text.
   - Inline skills: return a brief "Launching skill: X" acknowledgment (the real content is in `newMessages`).

#### Skill Discovery in the Prompt

The skill listing is budget-constrained to 1% of the context window (in characters). `prompt.ts::formatCommandsWithinBudget()` builds the listing:

- Bundled skills always get full descriptions (never truncated).
- Plugin/user skills are truncated proportionally if the total exceeds the budget.
- Each description is capped at 250 characters (`MAX_LISTING_DESC_CHARS`).
- In extreme cases, non-bundled skills fall back to names-only.

Skills are also surfaced in `system-reminder` messages in the conversation, which the model reads to know what skills are available.

#### UI Components

`tools/SkillTool/UI.tsx` provides React/Ink rendering functions for the skill tool's lifecycle:
- `renderToolUseMessage` -- Shows the skill being invoked
- `renderToolUseProgressMessage` -- Shows progress during forked execution
- `renderToolResultMessage` -- Shows the skill completion
- `renderToolUseRejectedMessage` -- Shows permission denial
- `renderToolUseErrorMessage` -- Shows errors

### 3.4 Built-in vs User Skills

| Aspect | Built-in (Bundled) | User / Plugin Skills |
|---|---|---|
| Location | `skills/bundled/*.ts` | `~/.claude/skills/`, `.claude/skills/`, plugin dirs |
| Format | TypeScript with `getPromptForCommand` | Markdown with YAML frontmatter |
| Registration | `registerBundledSkill()` at startup | Discovered from filesystem at load time |
| Prompt truncation | Never truncated in listing | Truncated proportionally to fit budget |
| Feature gates | Can be gated on feature flags | Always available |
| Reference files | Extracted lazily via `files` property | Read directly from disk |
| Permission | Auto-allowed (safe properties) | May require user permission |

The `/skills` slash command (`commands/skills/skills.tsx`) renders the `SkillsMenu` component, which shows all available skills grouped by source (user settings, project settings, plugin, MCP) with token estimates and file paths.

---

## 4. Extension Points -- How to Extend Claude Code

### 4.1 Create a Skill (Simplest)

Create a markdown file in `~/.claude/skills/` (user-global) or `.claude/skills/` (project-specific):

```markdown
---
description: Run the project test suite and fix failures
allowed-tools: Bash, Edit, Read
when-to-use: Use when the user asks to run tests or fix test failures
---

Run the test suite with `npm test`. If tests fail, analyze the output and fix the issues.
Use $ARGUMENTS if the user specified specific test files.
```

The skill is immediately available as a slash command. The name is derived from the filename (e.g., `run-tests.md` becomes `/run-tests`). Alternatively, create `skills/run-tests/SKILL.md` for skills that need companion files.

### 4.2 Create a Plugin

Create a directory with a `plugin.json` manifest:

```json
{
  "name": "my-plugin",
  "description": "My custom plugin",
  "version": "1.0.0",
  "skills": "./skills",
  "hooks": "./hooks.json",
  "agents": "./agents"
}
```

Install locally for development:
```bash
claude plugin install my-plugin --scope local
```

Or use `--plugin-dir /path/to/my-plugin` for session-only testing.

### 4.3 Create a Built-in Plugin

For features that should ship with the CLI but be user-toggleable:

1. Create a definition in `plugins/bundled/index.ts`.
2. Call `registerBuiltinPlugin()` with the `BuiltinPluginDefinition`.
3. The plugin appears in `/plugin` UI with `{name}@builtin` identifier.

### 4.4 Create a Bundled Skill

For skills compiled into the CLI:

1. Create `skills/bundled/myskill.ts` exporting a `registerMySkill()` function.
2. Call `registerBundledSkill()` with a `BundledSkillDefinition`.
3. Import and call the register function in `skills/bundled/index.ts`.

### 4.5 Publish to a Marketplace

Create a git repository with a `marketplace.json`:

```json
{
  "name": "my-marketplace",
  "description": "My plugin marketplace",
  "plugins": {
    "my-plugin": {
      "source": { "source": "github", "repo": "myorg/my-plugin" },
      "description": "What it does"
    }
  }
}
```

Users add it via `/plugin` > Manage Marketplaces > Add, or:
```bash
claude plugin marketplace add <git-url>
```

---

## 5. Key Source Files Reference

### Plugin System

| File | Purpose |
|---|---|
| `types/plugin.ts` | Core type definitions: `LoadedPlugin`, `BuiltinPluginDefinition`, `PluginError`, `PluginLoadResult` |
| `utils/plugins/schemas.ts` | Zod schemas for manifests, marketplace entries, plugin IDs; official name validation |
| `utils/plugins/pluginLoader.ts` | Plugin discovery, loading, manifest validation, versioned caching |
| `plugins/builtinPlugins.ts` | Built-in plugin registry (`registerBuiltinPlugin`, `getBuiltinPlugins`) |
| `plugins/bundled/index.ts` | Built-in plugin initialization (currently scaffolding) |
| `services/plugins/pluginOperations.ts` | Core install/uninstall/enable/disable/update operations (pure functions) |
| `services/plugins/pluginCliCommands.ts` | CLI wrappers for plugin operations with output and process management |
| `services/plugins/PluginInstallationManager.ts` | Background marketplace reconciliation and auto-refresh |
| `hooks/useManagePlugins.ts` | React hook for initial plugin load and refresh notification |
| `utils/plugins/loadPluginCommands.ts` | Loads skill/command markdown from plugin directories |
| `utils/plugins/loadPluginHooks.ts` | Converts plugin hook configs to native matchers |
| `utils/plugins/loadPluginAgents.ts` | Loads agent definitions from plugins |
| `utils/plugins/mcpPluginIntegration.ts` | Extracts and deduplicates MCP server configs from plugins |
| `utils/plugins/lspPluginIntegration.ts` | Extracts LSP server configs from plugins |
| `utils/plugins/loadPluginOutputStyles.ts` | Loads output style configurations from plugins |
| `utils/plugins/marketplaceManager.ts` | Marketplace data loading and caching |
| `utils/plugins/reconciler.ts` | Syncs declared vs materialized marketplaces |
| `utils/plugins/pluginDirectories.ts` | Plugin cache directory paths (supports cowork mode) |
| `utils/plugins/pluginIdentifier.ts` | Parses `name@marketplace` identifiers |
| `utils/plugins/pluginPolicy.ts` | Enterprise policy enforcement for plugins |
| `utils/plugins/pluginBlocklist.ts` | Delisted plugin detection and removal |
| `utils/plugins/pluginVersioning.ts` | Version computation for cached plugins |
| `utils/plugins/dependencyResolver.ts` | Inter-plugin dependency resolution |
| `utils/plugins/installedPluginsManager.ts` | V2 installed plugins tracking on disk |
| `utils/plugins/pluginOptionsStorage.ts` | Per-plugin user configuration storage |
| `utils/plugins/pluginAutoupdate.ts` | Automatic plugin update logic |
| `utils/plugins/refresh.ts` | `refreshActivePlugins()` -- full cache clear and reload |
| `utils/plugins/validatePlugin.ts` | Plugin validation utilities |
| `commands/plugin/plugin.tsx` | Entry point for `/plugin` slash command |
| `commands/plugin/PluginSettings.tsx` | Main plugin management UI component |
| `commands/plugin/ManagePlugins.tsx` | Enable/disable plugins UI |
| `commands/plugin/BrowseMarketplace.tsx` | Marketplace browsing UI |
| `commands/plugin/ManageMarketplaces.tsx` | Marketplace sources management UI |
| `commands/plugin/ValidatePlugin.tsx` | Plugin validation UI |
| `commands/plugin/PluginTrustWarning.tsx` | Security confirmation dialog |

### Skills System

| File | Purpose |
|---|---|
| `skills/bundledSkills.ts` | `BundledSkillDefinition` type, `registerBundledSkill()`, bundled skill registry, file extraction |
| `skills/bundled/index.ts` | `initBundledSkills()` -- registers all bundled skills at startup |
| `skills/bundled/*.ts` | Individual bundled skill implementations (verify, debug, simplify, etc.) |
| `skills/loadSkillsDir.ts` | File-based skill discovery, frontmatter parsing, `createSkillCommand()` |
| `skills/mcpSkillBuilders.ts` | Bridge between MCP servers and skill loading pipeline |
| `tools/SkillTool/SkillTool.ts` | `Skill` tool definition: validation, permissions, inline/forked execution |
| `tools/SkillTool/prompt.ts` | Skill listing generation with budget-constrained truncation |
| `tools/SkillTool/constants.ts` | `SKILL_TOOL_NAME = 'Skill'` |
| `tools/SkillTool/UI.tsx` | React/Ink rendering for skill tool lifecycle states |
| `commands/skills/skills.tsx` | `/skills` slash command -- renders SkillsMenu |
| `components/skills/SkillsMenu.tsx` | Interactive skill browser grouped by source |
| `utils/skills/skillChangeDetector.ts` | Filesystem watcher for hot-reloading skills on change |
