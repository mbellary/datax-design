# Layer 4 - Primary Use Cases

## Purpose

This layer answers: what does this system do for users and clients?

Each use case below is tied to concrete source evidence: the command or API
surface, the request/payload type, the server-side dispatcher or processor, the
storage/update path when visible in this layer, and the response or UI surface.

## Source Map

| Concern | Concrete source |
|---|---|
| CLI product commands | `codex-rs/cli/src/main.rs`, `Subcommand` |
| App-server API methods | `codex-rs/app-server/README.md`, API overview |
| App-server request dispatcher | `codex-rs/app-server/src/message_processor.rs`, `ClientRequest::*` match arms |
| App-server payload structs | `codex-rs/app-server-protocol/src/protocol/v2/*.rs` |
| Non-interactive exec/review | `codex-rs/exec/src/lib.rs` |
| Thread storage | `codex-rs/thread-store/README.md` |
| App-server daemon lifecycle | `codex-rs/app-server-daemon/README.md` |
| Exec-server process/filesystem API | `codex-rs/exec-server/README.md`, `codex-rs/exec-server-protocol/src/protocol.rs` |
| Cloud task execution | `codex-rs/cloud-tasks/src/lib.rs` |
| TypeScript SDK | `sdk/typescript/README.md` |

## Use Case Catalog

| # | Use case | Actor/surface | Primary request/command | Primary handler | Main result |
|---:|---|---|---|---|---|
| 1 | Interactive local coding session | Human in terminal | `codex [OPTIONS] [PROMPT]` with no subcommand | CLI forwards to TUI/in-process app-server | Streaming terminal UI, persisted thread |
| 2 | Non-interactive agent run | Human/script/SDK | `codex exec` | `codex-exec` starts in-process app-server | Final stdout or JSONL events |
| 3 | Non-interactive review | Human/script | `codex review` or app-server `review/start` | `turn_processor.review_start` | Review turn and review items |
| 4 | Rich client conversation | VS Code/desktop/mobile/custom app | `thread/start`, `turn/start` | `thread_processor`, `turn_processor` | Thread/turn objects plus item notifications |
| 5 | Resume/list/read stored work | Human/client | `resume`, `thread/list`, `thread/read`, `thread/items/list` | `thread_processor` | Stored threads, turns, items |
| 6 | Manage thread lifecycle metadata | Human/client | archive/delete/unarchive/name/metadata/goal APIs | `thread_processor`, `thread_goal_processor` | Stored metadata updated and notifications emitted |
| 7 | Run utility commands without a turn | Client/tool UI | `command/exec` | `command_exec_processor` | Buffered response or output delta notifications |
| 8 | Host filesystem operations for rich clients | Client/tool UI | `fs/readFile`, `fs/writeFile`, `fs/watch`, etc. | `fs_processor` | File bytes/metadata/events |
| 9 | Remote or separate execution environment | Harness/client | `codex exec-server`, `process/start`, `fs/*` | exec-server RPC router/process handlers | Process and filesystem RPC over websocket/relay |
| 10 | Keep app-server running remotely | Desktop/mobile/SSH user | `codex app-server daemon ...` | daemon lifecycle commands | Detached app-server, pid/settings files |
| 11 | Discover/use skills, plugins, apps | Rich client/human | `skills/list`, `plugin/list`, `marketplace/add`, `app/list` | catalog/plugin/apps processors | Catalog rows and installed plugin/app state |
| 12 | Call MCP tools/resources | Rich client/agent | `mcpServer/tool/call`, `mcpServer/resource/read` | `mcp_processor` | MCP tool/resource result |
| 13 | Authenticate and inspect account | Human/client | `login`, `logout`, account app-server APIs | CLI login flow or `account_processor` | Stored auth state, auth/account status |
| 14 | Run cloud tasks | Human in CLI | `codex cloud ...` | `codex-cloud-tasks` backend client | Task URL/list/apply flow |
| 15 | Embed Codex from TypeScript | Application developer | `@openai/codex-sdk` `startThread().run()` | SDK spawns CLI and reads JSONL | Structured turn object/events |

## 1. Interactive Local Coding Session

**Actor/surface:** a human runs `codex` in a terminal.

**Entrypoint:** `codex-rs/cli/src/main.rs` defines `MultitoolCli` and says that
if no subcommand is specified, options are forwarded to the interactive CLI.
The same file defines `Subcommand`; `Exec`, `Review`, `Login`, and the other
explicit modes are separate branches.

**Request/payload:** this path is not an external JSON-RPC request from the
user. It starts the terminal UI and uses app-server/client types internally.
Earlier layer evidence shows `codex-rs/tui/src/lib.rs` imports
`InProcessAppServerClient`, `RemoteAppServerClient`, and `AppServerSession`.

**Handler/routing:** the TUI is the UI/controller. It talks to an in-process or
remote app-server client, which eventually reaches the same app-server request
semantics as JSON clients.

**Storage/update:** active conversation items are persisted through the thread
storage boundary. `codex-thread-store` states that `ThreadStore::append_items`
is the canonical history append API, `ThreadStore::update_thread_metadata` is
the only thread metadata write API, and `LocalThreadStore` persists history via
rollout JSONL files plus metadata in SQLite when available.

**Surfaced output:** terminal UI events: streamed agent messages, tool calls,
file edits, command output, approvals, and final answers.

**Build-from-scratch note:** implement this after the core protocol and storage
exist. The terminal UI should be a client of your conversation engine, not the
place where model/session state is invented.

## 2. Non-Interactive Agent Run

**Actor/surface:** a human, shell script, CI job, or SDK runs `codex exec`.

**Entrypoint:** `codex-rs/cli/src/main.rs` maps `Subcommand::Exec(ExecCli)` to
"Run Codex non-interactively." `codex-rs/exec/src/lib.rs` is the implementation
crate. Its top comment is an output contract:

- default mode writes only the final message to stdout
- `--json` mode writes valid JSONL, one event per line
- all other output goes to stderr

**Request/payload:** `codex-rs/exec/src/lib.rs` imports app-server payloads such
as `ThreadStartParams`, `TurnStartParams`, `ThreadResumeParams`,
`TurnInterruptParams`, and `ReviewStartParams`. It also defines
`InitialOperation::UserTurn { items, output_schema }`.

**Handler/routing:** `codex-exec` starts an `InProcessAppServerClient`, sends
typed `ClientRequest` values instead of raw JSON, and processes
`InProcessServerEvent` values.

**Storage/update:** same as app-server thread/turn flow: a thread is created or
resumed, items are appended, and metadata is updated through thread-store.

**Surfaced output:** either one final stdout message or JSONL events. The
exported event types in `codex-rs/exec/src/lib.rs` include
`ThreadStartedEvent`, `TurnStartedEvent`, `ItemStartedEvent`,
`ItemCompletedEvent`, `TurnCompletedEvent`, `TurnFailedEvent`, `Usage`,
`CommandExecutionItem`, `FileChangeItem`, `McpToolCallItem`, and
`AgentMessageItem`.

**Build-from-scratch note:** make non-interactive mode a thin app-server client.
Do not duplicate conversation orchestration.

## 3. Non-Interactive Code Review

**Actor/surface:** a human or script runs `codex review`; a rich client can call
`review/start`.

**Entrypoint:** `codex-rs/cli/src/main.rs` maps `Subcommand::Review` to "Run a
code review non-interactively." App-server exposes `review/start`.

**Request/payload:** `codex-rs/app-server-protocol/src/protocol/v2/review.rs`
defines:

- `ReviewStartParams { thread_id, target, delivery }`
- `ReviewTarget::UncommittedChanges`
- `ReviewTarget::BaseBranch { branch }`
- `ReviewTarget::Commit { sha, title }`
- `ReviewTarget::Custom { instructions }`
- `ReviewStartResponse { turn, review_thread_id }`

**Handler/routing:** `codex-rs/app-server/src/message_processor.rs` dispatches
`ClientRequest::ReviewStart` to `turn_processor.review_start`.

**Storage/update:** review is represented as a turn. The app-server README says
`review/start` responds like `turn/start` and emits `item/started` and
`item/completed` notifications, including review-mode items and a final
assistant review message.

**Surfaced output:** non-interactive CLI review text or app-server stream
events plus final review turn.

**Build-from-scratch note:** review should be a specialized turn type with a
review target payload, not an unrelated command path.

## 4. Rich Client Conversation

**Actor/surface:** VS Code extension, desktop app, mobile app, or custom client.

**Entrypoint:** `codex app-server` exposes a JSON-RPC-like protocol. The README
states that it supports stdio JSONL, websocket frames, Unix socket websocket
upgrade, and `off`.

**Request/payload:** the core conversation calls are:

- `initialize` with client metadata
- `initialized` notification
- `thread/start`
- `thread/resume`
- `thread/fork`
- `turn/start`
- `turn/steer`
- `turn/interrupt`

The payload structs are concrete:

- `ThreadStartParams` in `protocol/v2/thread.rs`
- `TurnStartParams` in `protocol/v2/turn.rs`

`ThreadStartParams` includes fields such as model, model provider, cwd,
approval policy, sandbox, permissions profile, config, instructions,
personality, ephemeral flag, thread source, environments, dynamic tools, and
selected capability roots.

`TurnStartParams` includes thread id, client user message id, input,
additional context, environments, cwd, approval policy, sandbox policy,
permissions, model, service tier, reasoning effort, personality, and optional
output schema.

**Handler/routing:** `message_processor.process_request` deserializes JSON into
a `ClientRequest`, then delegates to `handle_client_request`. In that dispatch:

- `ClientRequest::ThreadStart` -> `thread_processor.thread_start`
- `ClientRequest::ThreadResume` -> `thread_processor.thread_resume`
- `ClientRequest::ThreadFork` -> `thread_processor.thread_fork`
- `ClientRequest::TurnStart` -> `turn_processor.turn_start`
- `ClientRequest::TurnSteer` -> `turn_processor.turn_steer`
- `ClientRequest::TurnInterrupt` -> `turn_processor.turn_interrupt`

**Storage/update:** thread creation/resume creates or resumes a stored thread.
Turns append persisted items through the thread-store path. Metadata changes are
not inferred inside storage; they are explicit updates above the store.

**Surfaced output:** request responses return thread/turn objects. Streaming
progress is sent as notifications such as `thread/started`, `turn/started`,
`item/started`, `item/agentMessage/delta`, `item/completed`, and
`turn/completed`.

**Build-from-scratch note:** this is the main API spine. Implement the
connection handshake, thread lifecycle, turn lifecycle, and streaming event bus
before adding plugins, cloud, or remote control.

## 5. Resume, List, and Read Stored Work

**Actor/surface:** TUI resume picker, CLI resume/archive/delete commands, rich
clients showing history.

**Entrypoint:** CLI subcommands include `Resume`, `Archive`, `Delete`,
`Unarchive`, and `Fork`. App-server exposes `thread/list`, `thread/read`,
`thread/turns/list`, and `thread/items/list`.

**Request/payload:** list/read requests are app-server `ClientRequest` variants
with corresponding params in the protocol crate. The README states that
`thread/list` supports cursor pagination and filters, `thread/read` can include
turns, `thread/turns/list` pages turn history, and `thread/items/list` pages
persisted thread items.

**Handler/routing:** `message_processor.rs` dispatches:

- `ThreadList` -> `thread_processor.thread_list`
- `ThreadRead` -> `thread_processor.thread_read`
- `ThreadTurnsList` -> `thread_processor.thread_turns_list`
- `ThreadItemsList` -> `thread_processor.thread_items_list`

**Storage/update:** read/list operations query the thread store and SQLite
metadata where available. They do not append new model-visible history.

**Surfaced output:** saved thread rows, turns, and items for pickers, history
views, resume flows, and SDK resume support.

**Build-from-scratch note:** model the read side separately from live sessions.
You need durable append-only history plus queryable metadata.

## 6. Manage Thread Lifecycle Metadata

**Actor/surface:** CLI, rich clients, desktop/mobile controls.

**Entrypoint:** app-server methods include `thread/archive`, `thread/delete`,
`thread/unarchive`, `thread/name/set`, `thread/metadata/update`,
`thread/goal/set`, `thread/goal/get`, `thread/goal/clear`, and
`thread/memoryMode/set`.

**Request/payload:** examples include `ThreadGoalSetParams` in
`protocol/v2/thread.rs` and metadata/config payloads in the same protocol area.

**Handler/routing:** `message_processor.rs` dispatches:

- `ThreadArchive`, `ThreadDelete`, `ThreadUnarchive`, `ThreadSetName`,
  `ThreadMetadataUpdate`, and `ThreadMemoryModeSet` to `thread_processor`
- `ThreadGoalSet`, `ThreadGoalGet`, and `ThreadGoalClear` to
  `thread_goal_processor`

**Storage/update:** this is the explicit metadata path. The thread-store README
says `ThreadStore::update_thread_metadata` is the only thread metadata write
API. Goal and memory state also use persisted state managed below the
processors.

**Surfaced output:** updated thread objects or empty success objects, plus
notifications such as `thread/archived`, `thread/deleted`,
`thread/unarchived`, `thread/name/updated`, `thread/goal/updated`, and
`thread/goal/cleared`.

**Build-from-scratch note:** keep "append conversation item" and "patch
metadata" separate. It prevents accidental history writes and makes history
replay predictable.

## 7. Run Utility Commands Without a Turn

**Actor/surface:** rich client or tool UI that needs a command result but does
not want a Codex conversation turn.

**Entrypoint:** app-server `command/exec`.

**Request/payload:** `CommandExecParams` in
`protocol/v2/command_exec.rs` includes command argv, optional process id, PTY
mode, streaming stdin/stdout flags, output cap, timeout, cwd, env overrides,
terminal size, and sandbox policy.

**Handler/routing:** `message_processor.rs` dispatches:

- `OneOffCommandExec` -> `command_exec_processor.one_off_command_exec`
- `CommandExecWrite` -> `command_exec_processor.command_exec_write`
- `CommandExecResize` -> `command_exec_processor.command_exec_resize`
- `CommandExecTerminate` -> `command_exec_processor.command_exec_terminate`

**Storage/update:** this does not create a thread or turn. It is a utility
execution surface. If a client wants the command result in conversation history,
it must separately add or start a turn.

**Surfaced output:** final command response or `command/exec/outputDelta`
notifications for streaming output.

**Build-from-scratch note:** separate utility command execution from
agent-visible tool execution. They have different storage and audit semantics.

## 8. Host Filesystem Operations For Rich Clients

**Actor/surface:** editor extension, desktop UI, or remote client.

**Entrypoint:** app-server filesystem methods:

- `fs/readFile`
- `fs/writeFile`
- `fs/createDirectory`
- `fs/getMetadata`
- `fs/readDirectory`
- `fs/remove`
- `fs/copy`
- `fs/watch`
- `fs/unwatch`

**Request/payload:** `protocol/v2/fs.rs` defines payloads such as:

- `FsReadFileParams { path }`
- `FsReadFileResponse { data_base64 }`
- `FsWriteFileParams { path, data_base64 }`
- `FsGetMetadataResponse { is_directory, is_file, is_symlink, created_at_ms, modified_at_ms }`

**Handler/routing:** `message_processor.rs` dispatches filesystem requests to
`fs_processor`, for example `ClientRequest::FsReadFile`.

**Storage/update:** writes update the host filesystem, not thread storage. File
watching surfaces filesystem changes as notifications.

**Surfaced output:** base64 file contents, metadata records, directory entries,
or `fs/changed` notifications.

**Build-from-scratch note:** define filesystem APIs with absolute paths and
base64 bytes at the wire boundary so clients do not depend on server-native
text encoding.

## 9. Remote Or Separate Execution Environment

**Actor/surface:** app-server, remote environment harness, or another process
that needs process/filesystem execution on a different host.

**Entrypoint:** `codex exec-server`.

**Request/payload:** `codex-rs/exec-server-protocol/src/protocol.rs` defines
method constants:

- `initialize`, `initialized`
- `process/start`, `process/read`, `process/write`, `process/signal`,
  `process/terminate`
- `process/output`, `process/exited`, `process/closed`
- `environment/info`
- `fs/readFile`, `fs/open`, `fs/readBlock`, `fs/writeFile`,
  `fs/createDirectory`, `fs/getMetadata`, `fs/canonicalize`,
  `fs/readDirectory`, `fs/walk`, `fs/remove`, `fs/copy`
- `http/request`, `http/request/bodyDelta`

`ExecParams` includes process id, argv, cwd as `PathUri`, env policy/env map,
TTY flag, stdin behavior, optional argv0, sandbox context, and managed-network
enforcement.

**Handler/routing:** the exec-server README says the crate owns transport,
protocol, filesystem handlers, and process handlers. Its lifecycle is:
initialize request, initialize response, initialized notification, then process
or filesystem RPCs.

**Storage/update:** process state is connection-scoped. The README says when a
websocket connection closes, remaining managed processes for that client
connection are terminated. Filesystem writes affect the executor host.

**Surfaced output:** JSON-RPC responses and notifications over local websocket
or remote Noise relay frames.

**Build-from-scratch note:** remote execution deserves its own protocol and
process. It isolates OS/process concerns from the conversation server.

## 10. Keep App-Server Running Remotely

**Actor/surface:** desktop/mobile remote client or SSH bootstrap flow.

**Entrypoint:** `codex app-server daemon ...` and top-level
`codex remote-control`.

**Request/payload:** daemon commands are shell commands, not app-server JSON-RPC
methods:

- `start`
- `restart`
- `enable-remote-control`
- `disable-remote-control`
- `stop`
- `version`
- `bootstrap --remote-control`

**Handler/routing:** the daemon crate backs machine-readable lifecycle commands
for remote clients.

**Storage/update:** daemon state is under `CODEX_HOME/app-server-daemon/`:

- `settings.json`
- `app-server.pid`
- `app-server-updater.pid`
- `daemon.lock`

`bootstrap` records settings, starts app-server as a detached process, and
launches a detached updater loop.

**Surfaced output:** each command writes exactly one JSON object to stdout with
backend/socket/version/running-server details.

**Build-from-scratch note:** the daemon should own lifecycle and pid/settings
files only. It should not become a second app-server implementation.

## 11. Discover And Use Skills, Plugins, Apps

**Actor/surface:** rich client plugin browser, skill picker, app connector UI,
or CLI plugin management.

**Entrypoint:** CLI includes `Plugin(PluginCli)` and `Mcp(McpCli)`.
App-server exposes `skills/list`, `skills/extraRoots/set`, `hooks/list`,
`marketplace/add`, `marketplace/remove`, `marketplace/upgrade`, `plugin/list`,
`plugin/installed`, `plugin/read`, `plugin/skill/read`, `plugin/install`,
`plugin/uninstall`, `skills/config/write`, and `app/list`.

**Request/payload:** examples:

- `MarketplaceAddParams` in `protocol/v2/plugin.rs`
- `PluginInstallParams` in `protocol/v2/plugin.rs`

**Handler/routing:** `message_processor.rs` dispatches:

- skills/hooks/model/collaboration/permissions catalog requests to
  `catalog_processor`
- marketplace requests to `marketplace_processor`
- plugin requests to `plugin_processor`
- app listing to `apps_processor`

**Storage/update:** marketplace add/remove/upgrade updates user marketplace
configuration and installed marketplace roots. Plugin install/uninstall updates
plugin cache/config state. Skill config writes update user-level skill config.

**Surfaced output:** marketplace/plugin/app catalog rows, plugin details,
installed state, app accessibility, and skill file change notifications.

**Build-from-scratch note:** design extensions as catalogs plus install/config
state. Keep discovery, installation, and runtime tool exposure separate.

## 12. Call MCP Tools And Resources

**Actor/surface:** rich clients and agent runtime using configured MCP servers.

**Entrypoint:** app-server MCP methods:

- `mcpServer/oauth/login`
- `config/mcpServer/reload`
- `mcpServerStatus/list`
- `mcpServer/resource/read`
- `mcpServer/tool/call`

**Request/payload:** `McpServerToolCallParams` in
`protocol/v2/mcp.rs` defines the tool-call request payload.

**Handler/routing:** `message_processor.rs` dispatches MCP requests to
`mcp_processor`, including `mcp_server_tool_call`.

**Storage/update:** OAuth login updates MCP auth state. Config reload reads
server config from disk and queues loaded threads to refresh on the next active
turn.

**Surfaced output:** MCP server status rows, resource contents, OAuth
completion notifications, and tool-call results.

**Build-from-scratch note:** treat MCP as an external tool provider behind a
manager. Do not let every feature call MCP servers directly.

## 13. Authenticate And Inspect Account

**Actor/surface:** terminal users and rich clients.

**Entrypoint:** CLI has `Login(LoginCommand)` and `Logout(LogoutCommand)`.
App-server has account methods including login, logout, account read, auth
status, rate limits, token usage, workspace messages, and add-credit nudge.

**Request/payload:** app-server v1/v2 account protocol types live under
`app-server-protocol/src/protocol/v2/account.rs` and related files. CLI login
uses the `codex-login` crate.

**Handler/routing:** `message_processor.rs` dispatches:

- `LoginAccount` -> `account_processor.login_account`
- `LogoutAccount` -> `account_processor.logout_account`
- `GetAccount`, `GetAuthStatus`, rate-limit and token-usage requests to
  `account_processor`

**Storage/update:** login/logout changes local authentication state managed by
the login/auth crates and platform credential/config mechanisms.

**Surfaced output:** CLI login/logout result, account/auth status responses,
and login completion/cancellation behavior for rich clients.

**Build-from-scratch note:** isolate authentication behind an auth manager used
by API clients, cloud tasks, exec-server remote registration, and app-server
account APIs.

## 14. Run Cloud Tasks

**Actor/surface:** terminal user using `codex cloud`.

**Entrypoint:** `codex-rs/cli/src/main.rs` maps `Cloud(CloudTasksCli)` to
`codex cloud` / `cloud-tasks`.

**Request/payload:** `codex-rs/cloud-tasks/src/lib.rs` initializes an HTTP
backend with base URL defaulting to `https://chatgpt.com/backend-api`, requires
ChatGPT login, resolves environment id, resolves git ref, reads prompt from an
argument or stdin, then calls `CloudBackend::create_task`.

**Handler/routing:** this is not app-server routed. The cloud-tasks CLI talks
to `codex_cloud_tasks_client::CloudBackend`.

**Storage/update:** cloud task state lives in the backend service. Locally, the
CLI reads auth, git branch/default branch, and prompt input.

**Surfaced output:** for exec-style cloud task creation, the CLI prints the
task URL.

**Build-from-scratch note:** cloud execution is an integration boundary. Build
local session execution first; add cloud as a backend client later.

## 15. Embed Codex From TypeScript

**Actor/surface:** application developer using `@openai/codex-sdk`.

**Entrypoint:** `sdk/typescript/README.md` says the TypeScript SDK wraps the
`codex` CLI from `@openai/codex`, spawns the CLI, and exchanges JSONL events
over stdin/stdout.

**Request/payload:** developer APIs include:

- `new Codex()`
- `codex.startThread()`
- `thread.run(promptOrStructuredInput)`
- `thread.runStreamed(...)`
- `codex.resumeThread(threadId)`

The SDK can pass structured input entries, output JSON schema, images, working
directory controls, environment overrides, and config overrides.

**Handler/routing:** SDK -> spawned CLI -> non-interactive JSONL/app-server
path.

**Storage/update:** threads are persisted in `~/.codex/sessions`, as documented
by the SDK resume section.

**Surfaced output:** buffered `turn.finalResponse`, structured `turn.items`, or
streamed events such as `item.completed` and `turn.completed`.

**Build-from-scratch note:** an SDK can start as a process wrapper around a
stable CLI protocol. You do not need an in-process language binding on day one.

## Cross-Cutting Request/Response Shapes

| Shape | Used by | Payload format |
|---|---|---|
| CLI command | terminal, scripts | command-line flags, stdin, stdout/stderr |
| App-server request | rich clients | JSON-RPC-like request with `method`, `id`, `params` |
| App-server notification | rich clients | JSON-RPC-like notification with `method`, `params` |
| In-process app-server request | TUI/exec | typed Rust `ClientRequest`, no JSON serialization |
| Exec-server RPC | remote execution | exec-server JSON-RPC envelope, websocket or relay |
| SDK stream | TypeScript SDK | JSONL events over child process stdin/stdout |
| Cloud backend call | cloud tasks CLI | HTTP API through `CloudBackend` |

## Minimal Implementation Order From These Use Cases

1. Define durable conversation models: thread, turn, item, metadata, usage.
2. Implement append-only thread storage plus metadata updates.
3. Implement the core session engine that can start/resume a thread and run one
   turn.
4. Add a non-interactive CLI that starts a thread, starts a turn, and prints
   final output or JSONL events.
5. Add a terminal UI as a client of the same engine.
6. Add app-server JSON-RPC transport around the same typed request handlers.
7. Add command/filesystem utility APIs.
8. Add exec-server only when remote/separate execution becomes necessary.
9. Add plugin/skill/MCP catalogs after the core conversation loop is stable.
10. Add daemon, remote control, SDKs, and cloud tasks as outer integration
    layers.

## Layer 4 Conclusion

The primary system is a persisted coding conversation engine. The product
surfaces are mostly clients of that engine:

- terminal UI and `codex exec` use typed in-process app-server paths
- rich clients use app-server JSON-RPC methods
- exec-server isolates process/filesystem execution
- daemon manages long-running app-server lifecycle
- plugins, skills, MCP, apps, cloud tasks, and SDKs extend or wrap the core
  thread/turn/item model

For a from-scratch implementation, the critical first milestone is not the UI.
It is the thread/turn/item protocol, append-only storage, turn runner, and event
stream. Everything else becomes a client, extension, or lifecycle wrapper around
that spine.
