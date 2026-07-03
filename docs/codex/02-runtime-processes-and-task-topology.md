# Runtime Processes and Task Topology

This section answers the process question directly: what is a real operating
system process, what is daemon-managed, and what is an in-process async task.

## Process/Task Map

| Runtime unit | Kind | Owner | Purpose |
| --- | --- | --- | --- |
| `codex` | OS process | CLI binary | Main user command. Runs TUI by default or dispatches subcommands. |
| `codex app-server` | OS process / server | `codex-app-server` | JSON-RPC server for rich clients such as editors and desktop/mobile flows. |
| `codex app-server daemon` | lifecycle manager | `codex-app-server-daemon` | Starts, stops, restarts, bootstraps, and configures app-server as a detached Unix process. |
| app-server outbound router | in-process Tokio task | `codex-app-server` | Routes responses/notifications to connection writers without blocking request processing. |
| app-server processor loop | in-process Tokio task | `codex-app-server` | Receives transport events and dispatches JSON-RPC messages. |
| `codex exec-server` | OS process / server | `codex-exec-server` | JSON-RPC process/filesystem server, including remote execution mode. |
| core session task | in-process Tokio task | `codex-core` | Drives a turn: regular chat, review, compaction, shell command, etc. |
| thread-store live writer | in-process storage object | `codex-thread-store` | Owns live persistence state for a thread. |

## `codex app-server`

`codex app-server` is a long-running server process. Its command-line entry
point accepts a transport endpoint with these supported forms:

- `stdio://`
- `unix://`
- `unix://PATH`
- `ws://IP:PORT`
- `off`

It also accepts a `session-source`, strict config mode, auth settings, and a
hidden `remote-control` flag.

At startup, `main` parses `AppServerArgs`, resolves loader/auth/runtime options,
and calls `run_main_with_transport_options`.

Evidence:

- `codex-rs/app-server/src/main.rs:21` defines `AppServerArgs`.
- `codex-rs/app-server/src/main.rs:25` documents supported `--listen`
  transports.
- `codex-rs/app-server/src/main.rs:61` is the app-server `main`.
- `codex-rs/app-server/src/main.rs:95` calls
  `run_main_with_transport_options`.

## App-Server In-Process Loops

Inside the app-server process, there are two important long-running async loops:

1. outbound loop
2. processor loop

The source code states this directly: `run_main_with_transport_options` uses a
processor loop for incoming JSON-RPC/request dispatch and an outbound loop for
potentially slow per-connection writes.

The outbound loop:

- receives `OutboundControlEvent::Opened`, `Closed`, and `DisconnectAll`
- maintains `outbound_connections`
- receives outgoing envelopes
- calls `route_outgoing_envelope`

The processor loop:

- owns a `MessageProcessor`
- tracks open connections
- listens for transport events
- dispatches requests, responses, notifications, and errors
- sends initialize notifications after a connection becomes initialized
- tracks remote-control status changes
- listens for created threads so initialized clients can attach listeners

Evidence:

- `codex-rs/app-server/src/lib.rs:147` describes the control-plane events.
- `codex-rs/app-server/src/lib.rs:149` says there are two loops/tasks.
- `codex-rs/app-server/src/lib.rs:850` through
  `codex-rs/app-server/src/lib.rs:874` shows the outbound router loop.
- `codex-rs/app-server/src/lib.rs:877` creates the processor task.
- `codex-rs/app-server/src/lib.rs:931` enters the processor loop's
  `tokio::select!`.
- `codex-rs/app-server/src/lib.rs:948` receives transport events.
- `codex-rs/app-server/src/lib.rs:1014` handles incoming messages.
- `codex-rs/app-server/src/lib.rs:1023` calls `processor.process_request`.
- `codex-rs/app-server/src/lib.rs:1086` calls `processor.process_response`.
- `codex-rs/app-server/src/lib.rs:1093` calls
  `processor.process_notification`.

## `codex app-server daemon`

The daemon is not the same thing as app-server. It manages app-server lifecycle.

The daemon stores state under `CODEX_HOME/app-server-daemon/` and uses:

- `settings.json`
- `app-server.pid`
- `app-server-updater.pid`
- `daemon.lock`

The code defines lifecycle commands:

- `Start`
- `Restart`
- `Stop`
- `Version`

The README also documents user commands:

- `codex app-server daemon start`
- `codex app-server daemon restart`
- `codex app-server daemon enable-remote-control`
- `codex app-server daemon disable-remote-control`
- `codex app-server daemon stop`
- `codex app-server daemon version`
- `codex app-server daemon bootstrap --remote-control`

Evidence:

- `codex-rs/app-server-daemon/README.md` documents daemon behavior and state.
- `codex-rs/app-server-daemon/src/lib.rs:33` defines `daemon.lock`.
- `codex-rs/app-server-daemon/src/lib.rs:34` defines `settings.json`.
- `codex-rs/app-server-daemon/src/lib.rs:35` defines the
  `app-server-daemon` state directory name.
- `codex-rs/app-server-daemon/src/lib.rs:37` defines `LifecycleCommand`.
- `codex-rs/app-server-daemon/src/lib.rs:56` defines lifecycle output.

## `codex exec-server`

`codex exec-server` is a separate JSON-RPC server for process and filesystem
operations. It has its own protocol crate, method constants, router, and client.

The exec-server protocol includes:

- handshake: `initialize`, `initialized`
- process APIs: `process/start`, `process/read`, `process/write`,
  `process/signal`, `process/terminate`
- process notifications: `process/output`, `process/exited`, `process/closed`
- filesystem APIs: `fs/readFile`, `fs/open`, `fs/readBlock`, `fs/close`,
  `fs/writeFile`, `fs/createDirectory`, `fs/getMetadata`, `fs/canonicalize`,
  `fs/readDirectory`, `fs/walk`, `fs/remove`, `fs/copy`
- environment API: `environment/info`
- executor-side HTTP API: `http/request`,
  `http/request/bodyDelta`

The exec-server router is string-keyed. It stores request and notification
handlers in maps and decodes params into typed payloads before invoking the
handler.

Evidence:

- `codex-rs/exec-server/README.md` describes `codex exec-server`.
- `codex-rs/exec-server-protocol/src/protocol.rs:16` through
  `codex-rs/exec-server-protocol/src/protocol.rs:42` defines method constants.
- `codex-rs/exec-server/src/rpc.rs:117` stores request routes.
- `codex-rs/exec-server/src/rpc.rs:118` stores notification routes.
- `codex-rs/exec-server/src/rpc.rs:138` registers typed request handlers.
- `codex-rs/exec-server/src/rpc.rs:202` registers typed notification handlers.
- `codex-rs/exec-server/src/rpc.rs:224` resolves a request route by method.

## Core Session Tasks

Inside a loaded session, Codex does not block the whole process while a turn is
running. `codex-core` represents turn work as a `SessionTask` and runs it on a
Tokio task.

The `SessionTask` trait defines:

- `kind`
- `span_name`
- `run`
- `abort`

`Session::spawn_task` aborts any previous task for the active turn, clears
connector selection, then delegates to `start_task`.

`Session::start_task`:

- wraps the task as `AnySessionTask`
- marks turn timing and token usage
- creates a `CancellationToken`
- emits turn-start lifecycle
- creates a tracing span
- calls `tokio::spawn`
- runs the task
- flushes rollout history before completing the turn
- calls `on_task_finished` if not cancelled

Evidence:

- `codex-rs/core/src/tasks/mod.rs:206` documents `SessionTask`.
- `codex-rs/core/src/tasks/mod.rs:214` defines the trait.
- `codex-rs/core/src/tasks/mod.rs:232` defines `run`.
- `codex-rs/core/src/tasks/mod.rs:245` defines `abort`.
- `codex-rs/core/src/tasks/mod.rs:313` implements task spawning on `Session`.
- `codex-rs/core/src/tasks/mod.rs:314` defines `spawn_task`.
- `codex-rs/core/src/tasks/mod.rs:325` defines `start_task`.
- `codex-rs/core/src/tasks/mod.rs:401` calls `tokio::spawn`.
- `codex-rs/core/src/tasks/mod.rs:414` flushes rollout before task completion.

## Practical Design Lesson

If implementing from scratch, separate these layers early:

1. CLI command dispatch: starts the right mode.
2. Server process: owns transports, connections, and request dispatch.
3. Daemon/lifecycle manager: owns detached process state and restart/update
   policy.
4. Core session runtime: owns active turns and background tasks.
5. Execution server: isolates remote/local process and filesystem operations.
6. Storage boundary: persists thread history and queryable metadata.
