# Layer 2 - Product Surface Inventory

This layer answers: **what are all the ways users, tools, editors, SDKs,
servers, plugins, and external systems interact with Codex?**

## Surface Map

| Surface | Primary user | Entry point | Wire/interface style | Evidence |
|---|---|---|---|---|
| Interactive CLI/TUI | Human in terminal | `codex` with no subcommand | Terminal UI backed by app-server client types | `codex-rs/cli/src/main.rs:91`, `codex-rs/cli/src/main.rs:116`, `codex-rs/tui/src/lib.rs:21`, `codex-rs/tui/src/lib.rs:24` |
| Non-interactive exec | Shell scripts, CI, SDK wrappers | `codex exec` / `codex e` | stdout final output or JSONL events | `codex-rs/cli/src/main.rs:125`, `codex-rs/exec/src/lib.rs:1`, `codex-rs/exec/src/lib.rs:13` |
| Non-interactive review | Developers and CI | `codex review` | CLI request that drives review flow through app-server protocol types | `codex-rs/cli/src/main.rs:129`, `codex-rs/exec/src/lib.rs:28`, `codex-rs/exec/src/lib.rs:171` |
| Login/auth management | Human or automation | `codex login`, `codex logout` | CLI commands; stores/reuses Codex auth | `codex-rs/cli/src/main.rs:132`, `codex-rs/cli/src/main.rs:135`, `sdk/python/README.md:31` |
| MCP configuration client | Human configuring tools | `codex mcp` | CLI management of external MCP servers | `codex-rs/cli/src/main.rs:138` |
| Codex as MCP server | MCP clients | `codex mcp-server` | MCP over stdio, JSON-RPC messages | `codex-rs/cli/src/main.rs:144`, `codex-rs/mcp-server/src/lib.rs:1`, `codex-rs/mcp-server/src/lib.rs:57` |
| App server | IDEs, desktop/mobile, rich clients | `codex app-server` | JSON-RPC-like bidirectional protocol over stdio, websocket, or Unix socket | `codex-rs/cli/src/main.rs:147`, `codex-rs/app-server/README.md:3`, `codex-rs/app-server/README.md:22`, `codex-rs/app-server/README.md:24` |
| App-server daemon | Remote clients over SSH/machine lifecycle | `codex app-server daemon ...` | Machine-readable JSON lifecycle command output | `codex-rs/app-server-daemon/README.md:6`, `codex-rs/app-server-daemon/README.md:17`, `codex-rs/app-server-daemon/README.md:29` |
| Remote control | Desktop/mobile controlling remote app-server | `codex remote-control` and app-server remote-control RPCs | Daemon bootstrap plus app-server JSON-RPC status/pairing methods | `codex-rs/cli/src/main.rs:150`, `codex-rs/app-server-daemon/README.md:95`, `codex-rs/app-server/README.md:222` |
| Desktop app launcher | Human on macOS/Windows | `codex app` | Platform-specific launcher/installer command | `codex-rs/cli/src/main.rs:153` |
| Standalone exec-server | App-server remote environments, harnesses | `codex exec-server` | Exec-specific JSON-RPC over local websocket or remote Noise relay | `codex-rs/cli/src/main.rs:196`, `codex-rs/exec-server/README.md:3`, `codex-rs/exec-server/README.md:19`, `codex-rs/exec-server/README.md:22` |
| Plugin marketplace and plugin management | Users and app clients | `codex plugin`, app-server plugin RPCs | CLI and app-server JSON-RPC methods | `codex-rs/cli/src/main.rs:141`, `codex-rs/app-server/README.md:213`, `codex-rs/app-server/README.md:216`, `codex-rs/app-server-protocol/src/protocol/common.rs:672` |
| Skills | Agent capability authors and app clients | app-server `skills/*` and bundled skill crates | App-server JSON-RPC plus local/remote skill markdown metadata | `codex-rs/app-server/README.md:210`, `codex-rs/app-server/README.md:230`, `codex-rs/app-server-protocol/src/protocol/common.rs:657` |
| Apps/connectors | App clients and plugins | app-server `app/list` | App-server JSON-RPC list/update notifications | `codex-rs/app-server/README.md:221`, `codex-rs/app-server-protocol/src/protocol/common.rs:732` |
| Filesystem/process utility APIs | Rich clients | app-server `fs/*`, `process/*`, `command/exec` | JSON-RPC request/response and streaming notifications | `codex-rs/app-server/README.md:181`, `codex-rs/app-server/README.md:186`, `codex-rs/app-server/README.md:192`, `codex-rs/app-server-protocol/src/protocol/common.rs:737` |
| Cloud tasks | Human using Codex Cloud tasks locally | `codex cloud` / `codex cloud-tasks` | CLI backed by ChatGPT backend API | `codex-rs/cli/src/main.rs:189`, `codex-rs/cloud-tasks/src/lib.rs:43`, `codex-rs/cloud-tasks/src/lib.rs:49`, `codex-rs/cloud-tasks/src/lib.rs:172` |
| TypeScript SDK | Node/Electron/tool authors | `@openai/codex-sdk` | Spawns `codex` CLI and exchanges JSONL over stdio | `sdk/typescript/README.md:3`, `sdk/typescript/README.md:5`, `sdk/typescript/README.md:34` |
| Python SDK | Python app and automation authors | `openai-codex` | Starts threads, runs turns, streams progress through SDK wrapper | `sdk/python/README.md:1`, `sdk/python/README.md:3`, `sdk/python/README.md:22`, `sdk/python/README.md:28` |
| npm CLI package | Node package users | `@openai/codex` binary `codex` | JavaScript package wrapper for the CLI binary | `codex-cli/package.json:2`, `codex-cli/package.json:6` |
| Maintenance commands | Human/operator | `completion`, `update`, `doctor`, `sandbox`, `apply`, session archive/delete/fork/resume | CLI subcommands | `codex-rs/cli/src/main.rs:157`, `codex-rs/cli/src/main.rs:160`, `codex-rs/cli/src/main.rs:163`, `codex-rs/cli/src/main.rs:166`, `codex-rs/cli/src/main.rs:176`, `codex-rs/cli/src/main.rs:180` |

## The Main Product Surfaces

### 1. Terminal Product: `codex`

The top-level executable is a multitool CLI. The parser explicitly says that
when no subcommand is specified, options are forwarded to the interactive CLI
(`codex-rs/cli/src/main.rs:91` through `codex-rs/cli/src/main.rs:120`).

The interactive surface is not a thin prompt loop. It imports and uses the TUI
crate through `TuiCli` (`codex-rs/cli/src/main.rs:116`), and the TUI crate wires
itself to app-server client implementations:

- `App` owns the terminal application object (`codex-rs/tui/src/lib.rs:21`).
- `AppServerSession` is the TUI session bridge (`codex-rs/tui/src/lib.rs:24`).
- `InProcessAppServerClient` and `RemoteAppServerClient` show that the TUI can
  talk to app-server in-process or remotely (`codex-rs/tui/src/lib.rs:28` and
  `codex-rs/tui/src/lib.rs:30`).

Implementation lesson: if building this from scratch, do not put all agent
logic directly in terminal widgets. Put agent/session behavior behind an
app-server-like API, then make the TUI one client of that API.

### 2. Scriptable Product: `codex exec` and `codex review`

`codex exec` is the non-interactive agent mode (`codex-rs/cli/src/main.rs:125`).
The `codex-exec` crate states a strict output rule: default stdout should only
contain the final message, while `--json` stdout must be valid JSONL
(`codex-rs/exec/src/lib.rs:1` through `codex-rs/exec/src/lib.rs:4`).

`codex review` is another non-interactive mode (`codex-rs/cli/src/main.rs:129`).
The exec crate imports app-server review protocol types such as
`ReviewStartParams` and `ReviewStartResponse` (`codex-rs/exec/src/lib.rs:28`
and `codex-rs/exec/src/lib.rs:29`), which means review is not a completely
separate architecture. It is a specific operation over the same thread/turn
machinery.

Implementation lesson: expose one human-friendly interactive client and one
machine-friendly non-interactive client over the same underlying session
protocol.

### 3. Rich Client Surface: `codex app-server`

`codex app-server` is the explicit integration surface for richer clients such
as the VS Code extension (`codex-rs/app-server/README.md:3`). Its protocol is
bidirectional JSON-RPC-like messaging (`codex-rs/app-server/README.md:22`) over
stdio, websocket, Unix socket, or disabled local transport
(`codex-rs/app-server/README.md:24` through
`codex-rs/app-server/README.md:29`).

The app-server surface exposes three top-level user-work primitives:

- Thread: a conversation.
- Turn: one exchange within that conversation.
- Item: persisted input/output within a turn.

Those are documented at `codex-rs/app-server/README.md:64` through
`codex-rs/app-server/README.md:72`.

The lifecycle is explicit:

1. `initialize`
2. `initialized`
3. `thread/start`, `thread/resume`, or `thread/fork`
4. `turn/start`
5. streamed item notifications
6. `turn/completed`

That sequence is documented at `codex-rs/app-server/README.md:74` through
`codex-rs/app-server/README.md:81`.

Implementation lesson: for rich clients, design a stable protocol around
conversation state and streaming events. The UI should subscribe to state
changes instead of polling arbitrary internal structs.

### 4. Remote Machine Lifecycle Surface: App-Server Daemon

The daemon exists so remote clients can manage `codex app-server` on machines
reached over SSH or similar remote control paths. The README says it backs
machine-readable lifecycle commands used by desktop and mobile apps
(`codex-rs/app-server-daemon/README.md:6` through
`codex-rs/app-server-daemon/README.md:9`).

It exposes lifecycle commands such as:

- `start`
- `restart`
- `enable-remote-control`
- `disable-remote-control`
- `stop`
- `version`
- `bootstrap --remote-control`

Those are listed at `codex-rs/app-server-daemon/README.md:17` through
`codex-rs/app-server-daemon/README.md:27`.

The daemon emits exactly one JSON object on successful command output
(`codex-rs/app-server-daemon/README.md:29` through
`codex-rs/app-server-daemon/README.md:32`). It persists state under
`CODEX_HOME/app-server-daemon/` (`codex-rs/app-server-daemon/README.md:106`
through `codex-rs/app-server-daemon/README.md:113`).

Implementation lesson: split long-running app protocol from machine lifecycle.
The daemon starts, stops, upgrades, and enables remote-control mode; app-server
handles actual agent requests.

### 5. Remote Execution Surface: `codex exec-server`

`codex exec-server` is a standalone JSON-RPC process for filesystem and process
control. Its README says it is a small JSON-RPC server for spawning and
controlling subprocesses through `codex-utils-pty`
(`codex-rs/exec-server/README.md:3` through
`codex-rs/exec-server/README.md:5`).

It provides:

- CLI entry point `codex exec-server`
- Rust client `ExecServerClient`
- shared request/response protocol module

Those are listed at `codex-rs/exec-server/README.md:7` through
`codex-rs/exec-server/README.md:11`.

Transport options include local websocket and remote mode with an environment
registry and rendezvous websocket (`codex-rs/exec-server/README.md:22` through
`codex-rs/exec-server/README.md:30`). The lifecycle is also explicit:
`initialize`, initialize response, `initialized`, then process/filesystem RPCs
(`codex-rs/exec-server/README.md:109` through
`codex-rs/exec-server/README.md:116`).

Implementation lesson: remote execution should be a separate service boundary.
Do not make the UI process directly own all remote PTY, filesystem, and process
state.

### 6. Extension Surface: MCP, Plugins, Skills, Apps

Codex has two MCP roles:

- It can manage external MCP servers through `codex mcp`
  (`codex-rs/cli/src/main.rs:138`).
- It can run as an MCP server through `codex mcp-server`
  (`codex-rs/cli/src/main.rs:144`).

The MCP server crate describes itself as a prototype MCP server
(`codex-rs/mcp-server/src/lib.rs:1`) and uses RMCP JSON-RPC message types for
incoming messages (`codex-rs/mcp-server/src/lib.rs:16` through
`codex-rs/mcp-server/src/lib.rs:18`). Its incoming message alias is
`JsonRpcMessage<ClientRequest, Value, ClientNotification>`
(`codex-rs/mcp-server/src/lib.rs:57`).

Plugins and skills are exposed through both CLI and app-server:

- `codex plugin` is a CLI subcommand (`codex-rs/cli/src/main.rs:141`).
- app-server exposes `skills/list` and `skills/extraRoots/set`
  (`codex-rs/app-server-protocol/src/protocol/common.rs:657` through
  `codex-rs/app-server-protocol/src/protocol/common.rs:665`).
- app-server exposes marketplace and plugin methods
  (`codex-rs/app-server-protocol/src/protocol/common.rs:672` through
  `codex-rs/app-server-protocol/src/protocol/common.rs:705`).
- app-server exposes `app/list`
  (`codex-rs/app-server-protocol/src/protocol/common.rs:732` through
  `codex-rs/app-server-protocol/src/protocol/common.rs:735`).

Implementation lesson: treat extensions as a catalog/discovery problem plus an
execution problem. The app-server API lists, reads, installs, and configures
extensions; lower-level MCP/runtime code connects to tools.

### 7. SDK Surfaces

The TypeScript SDK is not a second independent agent runtime. Its README states
that it wraps the `codex` CLI from `@openai/codex`, spawns the CLI, and
exchanges JSONL events over stdin/stdout (`sdk/typescript/README.md:3` through
`sdk/typescript/README.md:5`).

The TypeScript SDK exposes:

- `Codex`
- `startThread`
- `thread.run`
- `thread.runStreamed`
- structured output schemas
- local image inputs
- `resumeThread`

Those appear in `sdk/typescript/README.md:17` through
`sdk/typescript/README.md:105`.

The Python SDK exposes thread start, turn run, progress collection, and auth
helpers. Its README says it builds Python apps that start threads, run turns,
stream progress, and control workspace access (`sdk/python/README.md:1`
through `sdk/python/README.md:4`). Its quickstart uses `Codex`,
`thread_start`, and `thread.run` (`sdk/python/README.md:19` through
`sdk/python/README.md:29`).

Implementation lesson: SDKs can initially wrap the stable CLI/app-server
protocol rather than embedding the full agent core.

### 8. Packaging Surface

The npm CLI package is named `@openai/codex`
(`codex-cli/package.json:2`). It exposes a `codex` binary that points to
`bin/codex.js` (`codex-cli/package.json:6` through
`codex-cli/package.json:8`). This package is a distribution surface, not the
main agent implementation.

Implementation lesson: separate packaging wrappers from core runtime. The
wrapper should locate and launch the right platform binary; it should not
contain the agent architecture.

### 9. Cloud Task Surface

The CLI exposes `codex cloud` / `codex cloud-tasks`
(`codex-rs/cli/src/main.rs:189`). The cloud-tasks crate initializes a backend
against `https://chatgpt.com/backend-api` by default
(`codex-rs/cloud-tasks/src/lib.rs:49` through
`codex-rs/cloud-tasks/src/lib.rs:50`) and creates tasks through a
`CloudBackend` (`codex-rs/cloud-tasks/src/lib.rs:172` through
`codex-rs/cloud-tasks/src/lib.rs:180`).

Implementation lesson: cloud tasks are an integration with remote backend task
state, not the same thing as a local thread. Keep that boundary explicit.

## What You Need To Implement From Scratch

For a minimal system inspired by this codebase, implement these surfaces in
this order:

1. **Core session protocol**: define Thread, Turn, Item, request, response, and
   notification types.
2. **App-server API**: expose `initialize`, `thread/start`, `turn/start`, item
   notifications, and `turn/completed` over JSON-RPC or JSONL.
3. **Interactive terminal client**: build a TUI/CLI that talks to the API
   instead of owning agent internals directly.
4. **Non-interactive CLI**: add a scriptable `exec` mode with clean stdout and
   optional JSONL streaming.
5. **Persistence surface**: add thread list/read/resume/archive APIs.
6. **Process/filesystem execution surface**: add sandboxed command execution
   locally first; split remote execution into an exec-server later.
7. **Extension discovery**: add MCP/tool/plugin/skill catalogs after the core
   conversation and execution flows work.
8. **SDK wrappers**: wrap the CLI or app-server protocol for TypeScript/Python.
9. **Daemon/lifecycle manager**: add only when remote machine lifecycle becomes
   necessary.

The key design principle from this layer: **Codex is not one interface. It is a
core agent/session system exposed through multiple clients and service
boundaries.**
