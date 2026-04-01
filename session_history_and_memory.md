# Session Management, History, and Memory Systems

This document describes how Claude Code manages conversational state across sessions, maintains prompt history, compacts context when it grows too large, and persists memories for cross-session recall.

---

## 1. Overview

Claude Code's conversational state management consists of four interconnected systems:

- **Session Management** -- Each conversation is identified by a UUID-based session ID. Sessions are persisted as JSONL transcript files on disk, enabling resume, rewind, and cross-session search. Sessions are scoped to a project directory (derived from the working directory or git root).

- **History System** -- A lightweight prompt-history log (`~/.claude/history.jsonl`) records every user input. This powers up-arrow recall and `Ctrl+R` search. History is project-scoped and session-aware, with current-session entries surfaced first.

- **Context Compaction** -- When the token count of a conversation approaches the model's context window limit, Claude Code summarizes older messages and replaces them with a compact summary. Multiple strategies exist: traditional API-based summarization, session-memory-based compaction (SM-compact), reactive compaction, and microcompaction (stripping old tool results).

- **Memory System** -- A persistent, file-based memory system stored under `~/.claude/projects/<sanitized-path>/memory/`. Memories are typed (user, feedback, project, reference), indexed via a `MEMORY.md` entrypoint, and recalled by a Sonnet-powered relevance selector. A background extraction agent automatically writes memories from conversation content.

---

## 2. Session Management

### 2.1 Session Creation and Storage

Each Claude Code session is identified by a UUID (`SessionId`). The bootstrap process (`bootstrap/state.ts`) generates this ID at startup and exposes it via `getSessionId()`.

**Transcript storage path:**

```
~/.claude/projects/<sanitized-cwd>/<session-id>.jsonl
```

The project directory is computed by `getProjectDir()` in `utils/sessionStorage.ts`, which sanitizes the current working directory into a filesystem-safe name via `sanitizePath()`. Long paths (>200 characters) are truncated and appended with a hash suffix for uniqueness.

**Transcript format:** Each line in the JSONL file is a serialized `Entry` object. Entry types include:
- `user` -- User messages (prompts, tool results)
- `assistant` -- Model responses (text, tool calls)
- `attachment` -- Context attachments (CLAUDE.md content, file state, etc.)
- `system` -- System messages (compact boundaries, memory-saved notifications, hook results)

Messages are linked via a `parentUuid` chain. Progress messages are excluded from this chain to prevent fork-related orphaning bugs.

**Subagent transcripts** are stored in a subdirectory:
```
~/.claude/projects/<sanitized-cwd>/<session-id>/subagents/agent-<agent-id>.jsonl
```

Subagent metadata (agent type, worktree path, description) is persisted in a `.meta.json` sidecar file alongside the transcript.

### 2.2 Session Resume and Recovery

The `/resume` command (aliases: `/continue`) allows resuming a previous conversation.

**Source:** `commands/resume/resume.tsx`, `commands/resume/index.ts`

**Behavior:**
1. Loads session logs via `loadSameRepoMessageLogs()` (or `loadAllProjectsMessageLogs()` when "all projects" is toggled on).
2. Filters out the current session and sidechain sessions (`filterResumableSessions()`).
3. Presents a `LogSelector` UI with session metadata (first prompt, timestamp, model, custom title).
4. On selection, validates the session UUID, loads the full log if needed (`loadFullLog()` for lite logs), and checks for cross-project resume.
5. Cross-project conversations display a command to copy to clipboard rather than resuming directly. Same-repo worktree conversations can resume directly.

**Lite logs:** For performance, session listing uses a "lite" metadata read (`readHeadAndTail()`) that reads only the first and last 64KB of the file, extracting the first user prompt, session ID, model, timestamp, and custom title without parsing the full transcript.

**Transcript loading for resume:** `readTranscriptForLoad()` in `sessionStoragePortable.ts` performs a single forward chunked read (1MB chunks) with in-stream compact boundary detection. When a compact boundary is found, everything before it is discarded (unless it has a `preservedSegment`). Attribution snapshot entries are stripped and only the last one is kept.

**CLI resume flag:** Sessions can also be resumed via `claude --resume <session-id>` or `claude --continue` (resume the most recent session from `getLastSessionLog()`).

**Agentic session search:** The `agenticSessionSearch()` utility allows natural-language search across session transcripts.

### 2.3 Session Rewind

The `/rewind` command (alias: `/checkpoint`) restores the conversation and/or code to a previous point.

**Source:** `commands/rewind/rewind.ts`, `commands/rewind/index.ts`

This command opens a message selector (`context.openMessageSelector()`) that lets the user pick a checkpoint in the conversation to revert to. It returns a `skip` result to avoid appending any messages, as the state restoration is handled by the selector callback.

---

## 3. History System

### 3.1 Message History Structure

**Source:** `history.ts`

Prompt history is stored in `~/.claude/history.jsonl` as a global file shared across all projects. Each entry (`LogEntry`) contains:

| Field | Description |
|---|---|
| `display` | The display text shown in history (prompt text with paste references) |
| `pastedContents` | Pasted content records (inline for small content, hash references for large) |
| `timestamp` | Unix timestamp in milliseconds |
| `project` | The project root path at time of entry |
| `sessionId` | The session that created this entry |

**Paste content handling:**
- Content under 1024 bytes is stored inline in the history entry.
- Larger content is hashed and stored externally in a paste store, with only the hash reference kept in history.
- Images are filtered out (stored separately in an image cache).
- References in prompts use the format `[Pasted text #N +M lines]` or `[Image #N]`.

### 3.2 History Retrieval

**Up-arrow history** (`getHistory()`):
- Scoped to the current project directory.
- Current-session entries are yielded first, then other sessions' entries. This prevents concurrent sessions from interleaving history.
- Capped at `MAX_HISTORY_ITEMS` (100 entries).

**Ctrl+R search** (`getTimestampedHistory()`):
- Deduped by display text (newest first).
- Yields `TimestampedHistoryEntry` objects with lazy `resolve()` for paste content (the picker only needs display text and timestamp for the list).
- Also capped at 100 entries.

### 3.3 History Write Path

History writes are buffered and flushed asynchronously:
1. `addToHistory()` creates a `LogEntry` and pushes it to `pendingEntries`.
2. `flushPromptHistory()` acquires a file lock on `history.jsonl` and appends all pending entries.
3. On process exit, a cleanup handler ensures pending entries are flushed via `immediateFlushHistory()`.

**Undo support:** `removeLastFromHistory()` undoes the most recent add. Fast path: pops from the pending buffer. Slow path (entry already flushed): adds the timestamp to a `skippedTimestamps` set that the reader consults.

History is skipped entirely when `CLAUDE_CODE_SKIP_PROMPT_HISTORY` is set (e.g., in Tungsten verification sessions).

---

## 4. Context Compaction

### 4.1 When and Why Compaction Happens

As conversations grow, they approach the model's context window limit. Compaction reduces the message history while preserving essential information.

**Auto-compaction** triggers when:
- `isAutoCompactEnabled()` returns true (respects `DISABLE_COMPACT`, `DISABLE_AUTO_COMPACT` env vars, and user config `autoCompactEnabled`).
- Token count exceeds `getAutoCompactThreshold(model)`, which is `effectiveContextWindow - 13,000` buffer tokens.
- The query source is not a recursive agent (`session_memory`, `compact`, or context-collapse agent).

**Manual compaction** via `/compact [instructions]` triggers on user request.

**Circuit breaker:** After 3 consecutive auto-compact failures, further attempts are skipped for the session to prevent wasting API calls.

### 4.2 Compaction Strategies

Compaction is attempted in priority order:

#### 4.2.1 Session Memory Compaction (SM-Compact)

**Source:** `services/compact/sessionMemoryCompact.ts`

The preferred compaction method when session memory has been populated. Instead of calling the API to summarize, it:
1. Reads the session memory file (maintained by the `SessionMemory` background agent).
2. Calculates which messages to keep based on `lastSummarizedMessageId` and configurable thresholds (`minTokens: 10,000`, `minTextBlockMessages: 5`, `maxTokens: 40,000`).
3. Creates a compact boundary marker and summary message from the session memory content.
4. Preserves tool_use/tool_result pairs and thinking blocks by adjusting the keep index (`adjustIndexToPreserveAPIInvariants()`).

Gated behind `tengu_session_memory` and `tengu_sm_compact` feature flags.

#### 4.2.2 Traditional (API-Based) Compaction

**Source:** `services/compact/compact.ts`

Calls `compactConversation()`, which:
1. Runs pre-compact hooks (`executePreCompactHooks()`).
2. Builds a summarization prompt (see `services/compact/prompt.ts`) instructing the model to produce a detailed summary covering: primary request, key technical concepts, files/code, errors, user messages, pending tasks, current work, and next steps.
3. Runs the summary as a forked agent for prompt cache sharing.
4. Replaces the conversation with the summary message plus a compact boundary marker.
5. Runs post-compact hooks.

The summary prompt uses an `<analysis>` scratchpad block (stripped before insertion) and a `<summary>` block that becomes the new context.

#### 4.2.3 Reactive Compaction

**Source:** `services/compact/reactiveCompact.ts` (ant-only feature)

Triggered by the API's `prompt_too_long` error rather than proactively. Routes through the same summarization pipeline but fires only when the API actually rejects the request.

#### 4.2.4 Microcompaction

**Source:** `services/compact/microCompact.ts`

A lighter-weight strategy that runs before traditional compaction. It strips old tool result content from messages to reduce token count without full summarization. Only targets specific tools: `FileRead`, `Bash/Shell`, `Grep`, `Glob`, `WebSearch`, `WebFetch`, `FileEdit`, `FileWrite`.

Time-based microcompaction can also clear old tool results proactively (configurable via `TimeBasedMCConfig`).

### 4.3 Post-Compaction Cleanup

**Source:** `services/compact/postCompactCleanup.ts`

After compaction:
- User context caches are cleared.
- Compact warning state is suppressed.
- The prompt cache break detector is notified.
- `lastSummarizedMessageId` is reset (the old UUID no longer exists).
- Session start hooks re-run to restore CLAUDE.md and other context.

---

## 5. Memory System

### 5.1 Memory Directory Structure

**Source:** `memdir/memdir.ts`, `memdir/paths.ts`

The auto-memory directory lives at:
```
~/.claude/projects/<sanitized-git-root>/memory/
```

Path resolution order for `getAutoMemPath()`:
1. `CLAUDE_COWORK_MEMORY_PATH_OVERRIDE` env var (full-path override for Cowork)
2. `autoMemoryDirectory` in settings.json (trusted sources only: policy/local/user, NOT project settings for security)
3. Default: `<memoryBaseDir>/projects/<sanitized-git-root>/memory/`

All git worktrees of the same repo share one memory directory (via `findCanonicalGitRoot()`).

**Key files:**
- `MEMORY.md` -- The entrypoint index file. Always loaded into conversation context. Truncated at 200 lines or 25KB. Each entry should be one line under ~150 characters: `- [Title](file.md) -- one-line hook`.
- Individual memory files (e.g., `user_role.md`, `feedback_testing.md`) -- Contain the actual memory content with YAML frontmatter.

**Memory frontmatter format:**
```markdown
---
name: {{memory name}}
description: {{one-line description}}
type: {{user, feedback, project, reference}}
---

{{memory content}}
```

**Enable/disable:** Controlled by `isAutoMemoryEnabled()` which checks:
1. `CLAUDE_CODE_DISABLE_AUTO_MEMORY` env var
2. `CLAUDE_CODE_SIMPLE` (--bare mode)
3. CCR without persistent storage
4. `autoMemoryEnabled` in settings.json
5. Default: enabled

### 5.2 Memory Types

**Source:** `memdir/memoryTypes.ts`

Memories are constrained to four types capturing context NOT derivable from the current project state:

| Type | Description |
|---|---|
| `user` | Information about the user's role, goals, preferences, knowledge. Always private. |
| `feedback` | Guidance the user has given about how to approach work -- corrections AND confirmations. |
| `project` | Ongoing work context, goals, initiatives, bugs, incidents not derivable from code/git. |
| `reference` | Pointers to external systems (Linear projects, Grafana dashboards, Slack channels). |

**What NOT to save:** Code patterns, architecture, file paths, git history, debugging solutions, CLAUDE.md content, ephemeral task details. These exclusions apply even when the user explicitly asks.

### 5.3 Memory Recall (Finding Relevant Memories)

**Source:** `memdir/findRelevantMemories.ts`, `memdir/memoryScan.ts`

At query time, relevant memories are surfaced via a two-step process:

1. **Scan:** `scanMemoryFiles()` reads the memory directory recursively, parses frontmatter from each `.md` file (up to 30 lines), and returns a list of `MemoryHeader` objects (filename, path, mtime, description, type). Capped at 200 files, sorted newest-first.

2. **Select:** `selectRelevantMemories()` sends the query and a formatted memory manifest to Sonnet via `sideQuery()`. The model returns up to 5 filenames it deems relevant, using structured JSON output. Recently-used tools are passed so the selector avoids re-surfacing reference docs for tools already in active use.

**Staleness handling** (`memdir/memoryAge.ts`): Memories older than 1 day include a freshness caveat warning that claims about code, file paths, or line numbers may be outdated and should be verified.

### 5.4 Memory Extraction (Background Agent)

**Source:** `services/extractMemories/extractMemories.ts`, `services/extractMemories/prompts.ts`

A background agent automatically extracts durable memories from conversation transcripts.

**Lifecycle:**
1. Initialized once at startup via `initExtractMemories()`.
2. Runs at the end of each complete query loop (when the model produces a final response with no tool calls), registered via `handleStopHooks`.
3. Uses the forked agent pattern (`runForkedAgent`) for prompt cache sharing.
4. Skips extraction if the main agent already wrote to memory files (`hasMemoryWritesSince()`).
5. Configurable turn throttling via `tengu_bramble_lintel` (default: every eligible turn).

**Tool permissions:** The extraction agent has restricted access:
- **Allowed:** `FileRead`, `Grep`, `Glob`, read-only `Bash`, `REPL`
- **Write-restricted:** `FileEdit`/`FileWrite` only within the auto-memory directory
- **Denied:** All other tools (MCP, Agent, write-capable Bash, etc.)

**Prompt structure:** The extraction agent receives:
- The opener (message count, available tools, turn budget guidance, existing memory manifest)
- Memory type taxonomy (four-type system with examples)
- What NOT to save guidelines
- How to save instructions (frontmatter format, MEMORY.md index step)

**Overlap prevention:** If an extraction is already in progress when a new one is requested, the new context is stashed for a trailing run after the current one finishes.

### 5.5 Team Memory

**Source:** `memdir/teamMemPaths.ts`, `memdir/teamMemPrompts.ts`

When the `TEAMMEM` feature flag is enabled, a shared team memory directory exists as a subdirectory of the auto-memory directory:
```
~/.claude/projects/<sanitized-git-root>/memory/team/
```

Team memory adds:
- A `private` vs `team` scope dimension to each memory type.
- Per-type scope guidance (e.g., user memories are always private; project memories bias toward team).
- Path traversal security via `validateTeamMemWritePath()` and `validateTeamMemKey()` which resolve symlinks and check containment.
- Sensitive data prohibition for shared team memories.

### 5.6 Session Memory

**Source:** `services/SessionMemory/sessionMemory.ts`, `services/SessionMemory/prompts.ts`, `services/SessionMemory/sessionMemoryUtils.ts`

Session Memory is a within-session note-taking system that maintains a structured markdown file about the current conversation. Unlike the persistent memory system (cross-session), session memory captures ephemeral working context that powers SM-compact.

**Session memory file location:**
```
~/.claude/session-memory/<session-id>.md
```

**Template structure** (customizable at `~/.claude/session-memory/config/template.md`):
- Session Title
- Current State
- Task specification
- Files and Functions
- Workflow
- Errors and Corrections
- Codebase and System Documentation
- Learnings
- Key results
- Worklog

**Extraction triggers:** The background extraction hook (`extractSessionMemory`) fires when:
1. Context window tokens exceed the initialization threshold (default: 10,000 tokens).
2. Sufficient context growth since last extraction (default: 5,000 tokens).
3. Sufficient tool calls since last extraction (default: 3).
4. The last assistant turn has no pending tool calls (natural conversation break).

**Update mechanism:** Uses `runForkedAgent` with a prompt that instructs the model to use `FileEdit` on the session memory file. The model must preserve all section headers and italic descriptions, updating only the content below them. Each section is capped at ~2,000 tokens, with a total budget of 12,000 tokens.

**Configuration:** Remotely configurable via `tengu_sm_config` (GrowthBook). Custom prompts can be placed at `~/.claude/session-memory/config/prompt.md`.

### 5.7 The `/memory` Command

**Source:** `commands/memory/memory.tsx`, `commands/memory/index.ts`

The `/memory` command opens an interactive dialog (`MemoryFileSelector`) that lists all memory files from `getMemoryFiles()`. Selecting a file:
1. Creates the `~/.claude` directory if needed.
2. Creates the memory file if it does not exist.
3. Opens the file in the user's editor (`$VISUAL` or `$EDITOR`).

### 5.8 Memory Persistence Across Sessions

Memories persist across sessions through the file system:

1. **MEMORY.md entrypoint** is loaded into the system prompt at the start of every session via `loadMemoryPrompt()`. It is always in context.
2. **Individual memory files** are recalled on demand by the `findRelevantMemories()` selector when the user's query appears related.
3. **Session memory** survives compaction and powers SM-compact. If a session is resumed, session memory content is available even though `lastSummarizedMessageId` may be unset.
4. **Background extraction** continuously writes new memories as conversations progress, so they are available to future sessions.
5. **Assistant (KAIROS) mode** uses an append-only daily log pattern (`<memoryDir>/logs/YYYY/MM/YYYY-MM-DD.md`) instead of maintaining MEMORY.md live. A nightly `/dream` skill distills logs into topic files + MEMORY.md.

---

## 6. Key Source Files Reference

| File | Purpose |
|---|---|
| `history.ts` | Prompt history read/write, paste content handling, up-arrow and Ctrl+R support |
| `commands/session/session.tsx` | `/session` command -- shows remote session URL and QR code |
| `commands/session/index.ts` | Command registration for `/session` (alias: `/remote`) |
| `commands/resume/resume.tsx` | `/resume` command -- session picker UI, cross-project detection, resume logic |
| `commands/resume/index.ts` | Command registration for `/resume` (alias: `/continue`) |
| `commands/rewind/rewind.ts` | `/rewind` command -- opens message selector for checkpoint restore |
| `commands/rewind/index.ts` | Command registration for `/rewind` (alias: `/checkpoint`) |
| `commands/compact/compact.ts` | `/compact` command -- orchestrates compaction strategies |
| `commands/compact/index.ts` | Command registration for `/compact` |
| `commands/memory/memory.tsx` | `/memory` command -- file selector and editor integration |
| `commands/memory/index.ts` | Command registration for `/memory` |
| `memdir/memdir.ts` | Memory directory prompt building, MEMORY.md truncation, entrypoint loading |
| `memdir/paths.ts` | Auto-memory path resolution, enable/disable logic, path validation |
| `memdir/memoryTypes.ts` | Four-type taxonomy (user, feedback, project, reference), frontmatter format |
| `memdir/memoryAge.ts` | Memory age calculation and staleness caveats |
| `memdir/memoryScan.ts` | Memory directory scanning, frontmatter parsing, manifest formatting |
| `memdir/findRelevantMemories.ts` | Query-time memory recall via Sonnet-powered relevance selection |
| `memdir/teamMemPaths.ts` | Team memory paths, symlink-safe validation, containment checks |
| `memdir/teamMemPrompts.ts` | Combined auto+team memory prompt building |
| `services/extractMemories/extractMemories.ts` | Background memory extraction agent -- forked agent, tool permissions, overlap guard |
| `services/extractMemories/prompts.ts` | Extraction agent prompt templates (auto-only and combined variants) |
| `services/SessionMemory/sessionMemory.ts` | Session memory extraction hook, threshold logic, file setup |
| `services/SessionMemory/prompts.ts` | Session memory update prompt, template loading, section size analysis |
| `services/SessionMemory/sessionMemoryUtils.ts` | Session memory state tracking, config, threshold helpers |
| `services/compact/compact.ts` | Core compaction logic -- summarization, post-compact message building, cache sharing |
| `services/compact/autoCompact.ts` | Auto-compact trigger logic, token thresholds, circuit breaker |
| `services/compact/sessionMemoryCompact.ts` | SM-compact strategy -- message keep calculation, API invariant preservation |
| `services/compact/microCompact.ts` | Microcompaction -- strips old tool results from messages |
| `services/compact/prompt.ts` | Compaction summarization prompt templates |
| `services/compact/postCompactCleanup.ts` | Post-compaction cache clearing and state reset |
| `services/compact/compactWarningState.ts` | Context usage warning state management |
| `utils/sessionStorage.ts` | Transcript read/write, session file resolution, project directory management |
| `utils/sessionStoragePortable.ts` | Portable session utilities -- head/tail reads, path sanitization, chunked transcript loading |
