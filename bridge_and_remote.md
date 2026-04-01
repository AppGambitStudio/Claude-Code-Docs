# Bridge and Remote Execution System

## 1. Overview

The Bridge system enables Claude Code sessions to be controlled remotely from claude.ai (or other web clients). A local Claude Code CLI instance registers itself as a "bridge environment" with the Anthropic API, then polls for work dispatched by the server. When a user starts a session on claude.ai targeting that environment, the bridge spawns a child CLI process, connects it to the session-ingress transport layer, and relays messages bidirectionally.

There are two major modes:

- **Standalone bridge** (`claude remote-control`): A long-running daemon managed by `bridgeMain.ts` that polls for work, spawns child CLI sessions, and manages their lifecycle. Supports multi-session concurrency with configurable spawn modes (single-session, worktree, same-dir).
- **REPL bridge** (`/remote-control` slash command): An in-process bridge that connects an already-running REPL session to claude.ai, managed by `replBridge.ts` / `initReplBridge.ts`. The existing REPL becomes remotely controllable without spawning a separate process.

Two transport generations exist:

- **v1 (env-based)**: Uses the Environments API (`/v1/environments/bridge`) for registration and work dispatch. Transport is `HybridTransport` (WebSocket reads + HTTP POST writes to Session-Ingress).
- **v2 (env-less)**: Bypasses the Environments API. Connects directly via `POST /v1/code/sessions` and `POST /v1/code/sessions/{id}/bridge` to obtain a worker JWT. Transport is `SSETransport` (reads) + `CCRClient` (writes via `/worker/*` endpoints).

The v2 path is gated by the `tengu_bridge_repl_v2` GrowthBook flag and is REPL-only. Standalone/daemon modes remain on v1.

---

## 2. Architecture

### High-Level Component Map

```
claude.ai (Web UI)
     |
     | HTTPS / WebSocket
     v
Anthropic API  (Environments API, Sessions API, Session-Ingress, CCR v2)
     |
     | Poll / SSE / WebSocket
     v
+-------------------------------+
|  Bridge Layer (local CLI)     |
|                               |
|  bridgeMain.ts  (standalone)  |  <-- `claude remote-control`
|  replBridge.ts  (REPL)        |  <-- `/remote-control` command
|  remoteBridgeCore.ts (v2)     |  <-- env-less direct connect
|                               |
|  bridgeApi.ts   (HTTP client) |
|  bridgeMessaging.ts (protocol)|
|  replBridgeTransport.ts       |
|  sessionRunner.ts (spawner)   |
|  bridgeUI.ts    (terminal UI) |
+-------------------------------+
     |
     | stdio (JSON lines) or in-process
     v
+-------------------------------+
|  Child CLI / REPL Session     |
|  (runs tools, talks to LLM)  |
+-------------------------------+
```

### Component Responsibilities

| Component | Role |
|---|---|
| `bridgeMain.ts` | Standalone bridge loop: register environment, poll for work, spawn/manage sessions, handle backoff and reconnection |
| `replBridge.ts` | REPL bridge core (v1): poll-based work dispatch within an existing REPL process |
| `remoteBridgeCore.ts` | REPL bridge core (v2, env-less): direct `/bridge` JWT exchange, SSE+CCRClient transport |
| `initReplBridge.ts` | REPL entry point: gate checks, OAuth, git context, title derivation, then delegates to v1 or v2 core |
| `bridgeApi.ts` | HTTP client for the Environments API (`registerBridgeEnvironment`, `pollForWork`, `acknowledgeWork`, `stopWork`, `heartbeat`, `deregister`) |
| `bridgeMessaging.ts` | Shared message parsing, routing, dedup, type guards for `SDKMessage`, `SDKControlRequest`, `SDKControlResponse` |
| `replBridgeTransport.ts` | Transport abstraction (`ReplBridgeTransport`) with v1 (`createV1ReplTransport`) and v2 (`createV2ReplTransport`) factories |
| `sessionRunner.ts` | `createSessionSpawner` factory: spawns child CLI processes via `child_process.spawn`, parses their stdout JSON lines for activity tracking |
| `bridgeUI.ts` | Terminal UI: QR code, spinner, status lines, session list, reconnection display |
| `createSession.ts` | `createBridgeSession`: POST `/v1/sessions` to create a session on the server |
| `trustedDevice.ts` | Trusted device token enrollment, storage, and retrieval for elevated auth |
| `pollConfig.ts` | GrowthBook-backed poll interval configuration with Zod validation |
| `workSecret.ts` | Decode base64url work secrets, build SDK/CCR URLs, `registerWorker`, `sameSessionId` |
| `bridgeConfig.ts` | Auth/URL resolution: dev overrides (`CLAUDE_BRIDGE_*`) with OAuth fallback |

### Remote Directory (Client-Side Session Viewer)

The `remote/` directory contains the **client-side** components used when a local CLI connects to watch/interact with a remote session (the inverse direction of the bridge):

| Component | Role |
|---|---|
| `RemoteSessionManager.ts` | Coordinates WebSocket subscription + HTTP POST for sending/receiving messages to a remote CCR session |
| `SessionsWebSocket.ts` | WebSocket client for `/v1/sessions/ws/{id}/subscribe` with auth, reconnection, and ping keepalive |
| `sdkMessageAdapter.ts` | Converts `SDKMessage` (CCR wire format) to internal REPL `Message` types for local rendering |
| `remotePermissionBridge.ts` | Creates synthetic `AssistantMessage` and `Tool` stubs for permission prompts originating from the remote agent |

---

## 3. Session Lifecycle

### Standalone Bridge (`bridgeMain.ts`)

1. **Registration**: `createBridgeApiClient` calls `POST /v1/environments/bridge` with machine name, directory, branch, git URL, max sessions, and worker type. Returns `environment_id` and `environment_secret`.

2. **Poll loop** (`runBridgeLoop`): Continuously polls `pollForWork(environmentId, environmentSecret)`. The poll interval is configurable via GrowthBook (`tengu_bridge_poll_interval_config`), with defaults of 2s when seeking work and 10min when at capacity.

3. **Work dispatch**: When work arrives (a `WorkResponse` with `type: 'session'`), the bridge:
   - Decodes the `secret` field (base64url JSON) via `decodeWorkSecret` to extract `session_ingress_token`, `api_base_url`, `sources`, `auth`, and optional `claude_code_args`.
   - Builds an SDK URL via `buildSdkUrl` (v1) or `buildCCRv2SdkUrl` (v2).
   - Calls `acknowledgeWork` to confirm receipt.
   - Spawns a child CLI via `spawner.spawn(opts, dir)`.

4. **Session management**: Active sessions are tracked in `activeSessions: Map<string, SessionHandle>`. Each session has:
   - A timeout watchdog (`sessionTimers`) defaulting to `DEFAULT_SESSION_TIMEOUT_MS` (24 hours).
   - Activity tracking parsed from the child's stdout JSON lines.
   - Heartbeat via `heartbeatAllWork`.

5. **Session completion** (`onSessionDone`): Calls `stopWork` with retry, removes from active maps, cleans up worktrees if applicable, logs analytics. Status is one of `'completed' | 'failed' | 'interrupted'`.

6. **Spawn modes** (`SpawnMode`):
   - `single-session`: One session in cwd; bridge tears down when it ends.
   - `worktree`: Each session gets an isolated git worktree via `createAgentWorktree`.
   - `same-dir`: All sessions share cwd (can conflict).

### REPL Bridge (v1, `replBridge.ts`)

1. `initReplBridge` (in `initReplBridge.ts`) performs gate checks, OAuth validation, git context resolution, and title derivation.
2. Delegates to `initBridgeCore` (`replBridge.ts`) which registers an environment and enters a poll loop within the REPL process.
3. On work receipt, creates a `ReplBridgeTransport` (v1 or v2) and connects it. Messages flow directly between the transport and the REPL's message handlers.
4. Returns a `ReplBridgeHandle` with methods: `writeMessages`, `writeSdkMessages`, `sendControlRequest`, `sendControlResponse`, `sendResult`, `teardown`.

### REPL Bridge (v2, env-less, `remoteBridgeCore.ts`)

1. `initEnvLessBridgeCore` creates a session via `POST /v1/code/sessions` (no environment ID).
2. Fetches bridge credentials via `POST /v1/code/sessions/{id}/bridge` which returns a `worker_jwt`, `expires_in`, `api_base_url`, and `worker_epoch`. The `/bridge` call itself IS the worker registration (bumps epoch).
3. Creates a v2 transport via `createV2ReplTransport`.
4. Sets up a `createTokenRefreshScheduler` to proactively re-call `/bridge` before JWT expiry, yielding a new JWT + new epoch.
5. On 401 from SSE: rebuilds transport with fresh `/bridge` credentials, preserving the sequence number cursor.

---

## 4. Transport Layer

### Transport Abstraction (`ReplBridgeTransport`)

Defined in `replBridgeTransport.ts`, the `ReplBridgeTransport` interface provides:

```typescript
type ReplBridgeTransport = {
  write(message: StdoutMessage): Promise<void>
  writeBatch(messages: StdoutMessage[]): Promise<void>
  close(): void
  isConnectedStatus(): boolean
  getStateLabel(): string
  setOnData(callback: (data: string) => void): void
  setOnClose(callback: (closeCode?: number) => void): void
  setOnConnect(callback: () => void): void
  connect(): void
  getLastSequenceNum(): number
  readonly droppedBatchCount: number
  reportState(state: SessionState): void
  reportMetadata(metadata: Record<string, unknown>): void
  reportDelivery(eventId: string, status: 'processing' | 'processed'): void
  flush(): Promise<void>
}
```

### v1 Transport (`createV1ReplTransport`)

Wraps `HybridTransport` (from `cli/transports/HybridTransport.ts`), which combines:
- **WebSocket** for reading inbound messages from Session-Ingress.
- **HTTP POST** for writing outbound messages to Session-Ingress.

`getLastSequenceNum()` always returns 0 (v1 doesn't use SSE sequence numbers). State reporting and delivery tracking are no-ops.

### v2 Transport (`createV2ReplTransport`)

Wraps `SSETransport` (reads) + `CCRClient` (writes). Key setup:

1. Calls `registerWorker(sessionUrl, ingressToken)` to obtain `worker_epoch` (unless epoch is pre-supplied from `/bridge`).
2. Derives the SSE stream URL: `{sessionUrl}/worker/events/stream`.
3. Creates `SSETransport` with optional `initialSequenceNum` for resume-from-last-position.
4. Creates `CCRClient` with `worker_epoch` for authenticated writes to `/worker/*` endpoints.

Auth for v2 uses per-instance JWT (`ingressToken`), not OAuth. The JWT has session-scoped claims (`session_id`, `role=worker`). Multi-session callers pass `getAuthToken` closures for per-instance auth isolation.

### Sequence Number Carryover

When swapping transports (e.g., on proactive JWT refresh), the bridge reads `getLastSequenceNum()` from the old transport and passes it as `initialSequenceNum` to the new transport. This prevents the server from replaying the entire session history from seq 0.

### Polling and Backoff (`pollConfig.ts`, `pollConfigDefaults.ts`)

Poll intervals are configurable via the `tengu_bridge_poll_interval_config` GrowthBook flag. Validated with Zod schema requiring:
- `poll_interval_ms_not_at_capacity`: min 100ms, default 2000ms
- `poll_interval_ms_at_capacity`: 0 (disabled) or >= 100ms, default 600000ms (10 min)
- `non_exclusive_heartbeat_interval_ms`: min 0, default 0
- Multisession variants for `not_at_capacity`, `partial_capacity`, `at_capacity`
- Safety refinement: at least one of heartbeat or at-capacity poll must be enabled to prevent tight-loop polling.

The standalone bridge (`bridgeMain.ts`) uses exponential backoff with configurable parameters:

```typescript
type BackoffConfig = {
  connInitialMs: number    // 2000
  connCapMs: number        // 120000 (2 min)
  connGiveUpMs: number     // 600000 (10 min)
  generalInitialMs: number // 500
  generalCapMs: number     // 30000
  generalGiveUpMs: number  // 600000 (10 min)
  shutdownGraceMs?: number // 30000 (SIGTERM -> SIGKILL)
  stopWorkBaseDelayMs?: number // 1000
}
```

Sleep detection uses a threshold of `2 * connCapMs` to distinguish system sleep/wake from normal backoff delays.

---

## 5. Bridge Messaging Protocol

### Message Types (`bridgeMessaging.ts`)

The bridge handles three categories of wire messages:

1. **`SDKMessage`** -- The standard message union (discriminated on `type`). Includes `user`, `assistant`, `system`, `result`, `stream_event`, `status`, `compact_boundary`, etc.

2. **`SDKControlRequest`** -- Server-to-worker requests requiring prompt response (10-14s server timeout). Subtypes include `initialize`, `set_model`, `can_use_tool`.

3. **`SDKControlResponse`** -- Worker-to-server responses, or server-to-worker permission decisions. Contains a `response` field with the decision payload.

### Type Guards

```typescript
function isSDKMessage(value: unknown): value is SDKMessage
function isSDKControlResponse(value: unknown): value is SDKControlResponse
function isSDKControlRequest(value: unknown): value is SDKControlRequest
```

### Eligibility Filter (`isEligibleBridgeMessage`)

Only certain REPL messages are forwarded to the bridge transport:
- `user` messages (non-virtual)
- `assistant` messages (non-virtual)
- `system` messages with `subtype === 'local_command'`

Virtual messages, tool results, progress events, and other internal REPL chatter are filtered out.

### Ingress Routing (`handleIngressMessage`)

Parses inbound WebSocket/SSE data and routes to the appropriate handler:

1. JSON parse with `normalizeControlMessageKeys` for backward compatibility.
2. `control_response` -> `onPermissionResponse` callback.
3. `control_request` -> `onControlRequest` callback.
4. `SDKMessage` with `type === 'user'` -> `onInboundMessage` callback (with UUID dedup).
5. Non-user SDKMessages are logged and ignored.

### Echo Deduplication (`BoundedUUIDSet`)

Two bounded UUID sets prevent message duplication:
- `recentPostedUUIDs`: Tracks UUIDs of messages the bridge sent, to ignore server echoes.
- `recentInboundUUIDs`: Tracks UUIDs of messages already forwarded, to handle server re-deliveries after transport swaps.

### Title Extraction (`extractTitleText`)

Extracts title-worthy text from user messages for auto-titling sessions. Filters out:
- Non-user messages, meta messages (nudges), tool results, compact summaries.
- Non-human origins (task notifications, channel messages).
- Pure display-tag content (`<ide_opened_file>`, `<session-start-hook>`, etc.).

### Result Message (`makeResultMessage`)

Constructs a `result` SDKMessage with `subtype: 'success'` and empty usage, sent when a session completes.

---

## 6. Trusted Device Management

### Purpose

Bridge sessions have `SecurityTier=ELEVATED` on the server (CCR v2). The trusted device system adds a second authentication factor via `X-Trusted-Device-Token` header on bridge API calls.

### Implementation (`trustedDevice.ts`)

**Gate**: `tengu_sessions_elevated_auth_enforcement` (GrowthBook). Two-phase rollout: CLI-side gate (headers start flowing) then server-side gate (enforcement enabled).

**Enrollment** (`enrollTrustedDevice`):
1. Called immediately after `/login` (server requires `account_session.created_at < 10min`).
2. `POST /api/auth/trusted_devices` with `display_name: "Claude Code on {hostname} . {platform}"`.
3. Receives `device_token` and `device_id`.
4. Persists `device_token` to OS keychain via `getSecureStorage().update()`.
5. Clears the memoization cache so subsequent reads pick up the new token.

**Retrieval** (`getTrustedDeviceToken`):
1. Checks the GrowthBook gate (live, not cached).
2. Reads token from `CLAUDE_TRUSTED_DEVICE_TOKEN` env var (testing/canary override) or keychain.
3. Memoized via lodash `memoize` -- the keychain read spawns a macOS `security` subprocess (~40ms per call), and `bridgeApi.ts` calls this on every poll/heartbeat/ack.

**Cache Management**:
- `clearTrustedDeviceTokenCache()`: Clears the memoization cache (called after enrollment).
- `clearTrustedDeviceToken()`: Removes the token from keychain and clears cache (called on account switch to prevent cross-account token leakage).

---

## 7. REPL Bridge

### Entry Point (`initReplBridge.ts`)

`initReplBridge(options?: InitBridgeOptions)` is the public API called by `useReplBridge` (auto-start) and `print.ts` (SDK `-p` mode):

1. **Gate check**: `isBridgeEnabledBlocking()` (GrowthBook `tengu_bridge_repl`).
2. **OAuth check**: `getBridgeAccessToken()` must return a valid token.
3. **Policy check**: `waitForPolicyLimitsToLoad()` then `isPolicyAllowed('remote_control')`.
4. **v1/v2 branch**: `isEnvLessBridgeEnabled()` gates the v2 path. v2 calls `initEnvLessBridgeCore` from `remoteBridgeCore.ts`; v1 calls `initBridgeCore` from `replBridge.ts`.
5. **Title derivation**: Uses `initialName` if provided, else generates from `generateShortWordSlug` or derives via `generateSessionTitle` from conversation text.

### Handle Interface (`ReplBridgeHandle`)

```typescript
type ReplBridgeHandle = {
  bridgeSessionId: string
  environmentId: string
  sessionIngressUrl: string
  writeMessages(messages: Message[]): void
  writeSdkMessages(messages: SDKMessage[]): void
  sendControlRequest(request: SDKControlRequest): void
  sendControlResponse(response: SDKControlResponse): void
  sendControlCancelRequest(requestId: string): void
  sendResult(): void
  teardown(): Promise<void>
}
```

### State Machine (`BridgeState`)

```
'ready' -> 'connected' -> 'reconnecting' -> 'connected'
                                          -> 'failed'
```

State changes fire `onStateChange(state, detail?)` callbacks.

### Initial Message Flush

On connect, the bridge replays existing REPL conversation history to the server (capped by `initialHistoryCap`, default 200 messages). Only eligible messages (`isEligibleBridgeMessage`) are sent. UUIDs of previously flushed messages (from prior bridge sessions) are excluded via `previouslyFlushedUUIDs` to prevent cross-session UUID collisions.

### Inbound Message Flow

1. SSE/WebSocket frame arrives.
2. `handleIngressMessage` parses, deduplicates, and routes.
3. `user` messages fire `onInboundMessage` -> REPL processes as a new user turn.
4. `control_request` with `subtype: 'can_use_tool'` fires `handleServerControlRequest` -> `onSetPermissionMode` or tool permission response.
5. `control_response` fires `onPermissionResponse` -> resolves a pending permission prompt.

---

## 8. Bridge UI

### Terminal Display (`bridgeUI.ts`)

`createBridgeLogger(options)` returns a `BridgeLogger` object that manages the terminal display for the standalone bridge.

**Features**:
- **QR code**: Generated via the `qrcode` library with UTF-8 small format. Displays the connect URL for mobile scanning.
- **Spinner**: `BRIDGE_SPINNER_FRAMES` animation during the connecting phase (150ms per frame).
- **Status lines**: Live-updating status block at the bottom of the terminal, managed via ANSI escape sequences (`\x1b[{N}A` cursor up, `\x1b[J` erase to end).
- **Visual line counting**: `countVisualLines` accounts for terminal wrapping (`process.stdout.columns`).
- **Session list**: In multi-session mode, displays a bullet list of active sessions with titles, URLs, and current activity (tool starts, text output).

**State machine** (`StatusState`):
- `idle`: Ready for work, showing connect URL.
- `active`: Session(s) running, showing activity.
- `reconnecting`: Connection lost, showing retry status.
- `failed`: Permanent failure.

**Status line rendering** (`renderStatusLine`):
- Clears previous status lines.
- Renders QR code (if toggled visible).
- Renders session count indicator (`[2/4 sessions]`).
- Renders per-session activity (tool summaries with file paths, auto-expires via `TOOL_DISPLAY_EXPIRY_MS`).
- Footer with connect URL (OSC8 hyperlink for clickable terminals).

### Activity Tracking (`sessionRunner.ts`)

The session spawner parses child CLI stdout as JSON lines to extract activity:

- `assistant` messages with `tool_use` blocks -> `tool_start` activity with human-readable summary (e.g., "Reading src/foo.ts", "Running command").
- `assistant` messages with `text` blocks -> `text` activity (first 80 chars).
- `result` messages -> `result` or `error` activity.

Tool verbs are mapped via `TOOL_VERBS`:
```typescript
const TOOL_VERBS: Record<string, string> = {
  Read: 'Reading', Write: 'Writing', Edit: 'Editing',
  Bash: 'Running', Glob: 'Searching', Grep: 'Searching',
  WebFetch: 'Fetching', Task: 'Running task', ...
}
```

---

## 9. Key Source Files Reference

| File | Path | Purpose |
|---|---|---|
| `bridgeMain.ts` | `bridge/bridgeMain.ts` | Standalone bridge: `runBridgeLoop`, session management, backoff, multi-session spawn |
| `replBridge.ts` | `bridge/replBridge.ts` | REPL bridge core (v1 env-based): `initBridgeCore`, `ReplBridgeHandle`, `BridgeState` |
| `remoteBridgeCore.ts` | `bridge/remoteBridgeCore.ts` | REPL bridge core (v2 env-less): `initEnvLessBridgeCore`, direct `/bridge` JWT flow |
| `initReplBridge.ts` | `bridge/initReplBridge.ts` | REPL entry point: gate checks, OAuth, git context, v1/v2 dispatch |
| `bridgeApi.ts` | `bridge/bridgeApi.ts` | Environments API HTTP client: `createBridgeApiClient`, `BridgeFatalError` |
| `bridgeMessaging.ts` | `bridge/bridgeMessaging.ts` | Message parsing, routing, dedup: `handleIngressMessage`, `isEligibleBridgeMessage`, `extractTitleText` |
| `replBridgeTransport.ts` | `bridge/replBridgeTransport.ts` | Transport abstraction: `ReplBridgeTransport`, `createV1ReplTransport`, `createV2ReplTransport` |
| `sessionRunner.ts` | `bridge/sessionRunner.ts` | Child CLI spawner: `createSessionSpawner`, activity extraction, `TOOL_VERBS` |
| `bridgeUI.ts` | `bridge/bridgeUI.ts` | Terminal UI: `createBridgeLogger`, QR code, spinner, status lines |
| `createSession.ts` | `bridge/createSession.ts` | Session creation: `createBridgeSession` (POST `/v1/sessions`), `getBridgeSession` |
| `trustedDevice.ts` | `bridge/trustedDevice.ts` | Trusted device auth: `enrollTrustedDevice`, `getTrustedDeviceToken`, keychain storage |
| `pollConfig.ts` | `bridge/pollConfig.ts` | Poll interval config: `getPollIntervalConfig`, Zod validation, GrowthBook integration |
| `pollConfigDefaults.ts` | `bridge/pollConfigDefaults.ts` | Default poll intervals: `PollIntervalConfig` type, constant defaults |
| `workSecret.ts` | `bridge/workSecret.ts` | Work secret decode, URL builders: `decodeWorkSecret`, `buildSdkUrl`, `buildCCRv2SdkUrl`, `registerWorker`, `sameSessionId` |
| `bridgeConfig.ts` | `bridge/bridgeConfig.ts` | Auth/URL resolution: `getBridgeAccessToken`, `getBridgeBaseUrl`, dev overrides |
| `types.ts` | `bridge/types.ts` | Shared types: `BridgeConfig`, `WorkResponse`, `WorkSecret`, `SessionHandle`, `SpawnMode`, `BridgeApiClient` |
| `sessionIdCompat.ts` | `bridge/sessionIdCompat.ts` | Session ID conversion: `toCompatSessionId`, `toInfraSessionId` (session_* vs cse_* prefixes) |
| `jwtUtils.ts` | `bridge/jwtUtils.ts` | JWT refresh scheduler: `createTokenRefreshScheduler` for proactive token renewal |
| `capacityWake.ts` | `bridge/capacityWake.ts` | Signal to wake at-capacity sleep early when a session completes |
| `flushGate.ts` | `bridge/flushGate.ts` | Coordination gate for initial message flush completion |
| `bridgeDebug.ts` | `bridge/bridgeDebug.ts` | Fault injection and debug handles for testing |
| `envLessBridgeConfig.ts` | `bridge/envLessBridgeConfig.ts` | Config for env-less (v2) bridge: `getEnvLessBridgeConfig`, `EnvLessBridgeConfig` |
| `bridgeStatusUtil.ts` | `bridge/bridgeStatusUtil.ts` | Status formatting utilities: `buildBridgeConnectUrl`, `formatDuration`, `StatusState` |
| `RemoteSessionManager.ts` | `remote/RemoteSessionManager.ts` | Client-side remote session coordinator: WebSocket + HTTP POST, permission flow |
| `SessionsWebSocket.ts` | `remote/SessionsWebSocket.ts` | WebSocket client for `/v1/sessions/ws/{id}/subscribe`, reconnection, ping keepalive |
| `sdkMessageAdapter.ts` | `remote/sdkMessageAdapter.ts` | SDK-to-REPL message converter: `convertAssistantMessage`, `convertStreamEvent`, `convertResultMessage` |
| `remotePermissionBridge.ts` | `remote/remotePermissionBridge.ts` | Synthetic message/tool stubs for remote permission requests |
