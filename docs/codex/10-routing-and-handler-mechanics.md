# Layer 8 - Routing and Handler Mechanics

This layer explains how requests are routed to handlers, how responses are
correlated back to callers, which requests are serialized, and how internal
runtime operations are dispatched after a request crosses the public API
boundary.

This is not a generic flow summary. It is a routing map: entry point, envelope,
typed request, dispatcher, handler owner, concurrency gate, response path, and
late notification path.

## Source Map

| Area | Concrete source |
|---|---|
| CLI subcommand routing | `codex-rs/cli/src/main.rs` |
| App-server transport events | `codex-rs/app-server-transport/src/transport/mod.rs` |
| App-server process loops | `codex-rs/app-server/src/lib.rs` |
| App-server request deserialization and dispatch | `codex-rs/app-server/src/message_processor.rs` |
| App-server request serialization gates | `codex-rs/app-server/src/request_serialization.rs` |
| App-server request enum and method metadata | `codex-rs/app-server-protocol/src/protocol/common.rs` |
| App-server outgoing responses, notifications, server requests | `codex-rs/app-server/src/outgoing_message.rs` |
| In-process app-server runtime | `codex-rs/app-server/src/in_process.rs` |
| Core operation dispatch | `codex-rs/core/src/session/handlers.rs` |
| Core task execution | `codex-rs/core/src/tasks/mod.rs` |
| Exec-server router | `codex-rs/exec-server/src/rpc.rs` |
| Exec-server route registration | `codex-rs/exec-server/src/server/registry.rs` |

## Routing Layers

Codex has several routing layers. They are deliberately separate.

| Layer | What it routes | Main owner |
|---|---|---|
| CLI router | `codex ...` subcommands to product surfaces | `codex-rs/cli/src/main.rs` |
| Transport router | bytes or in-memory messages to connection-scoped events | `codex-rs/app-server-transport` and `app-server/src/in_process.rs` |
| App-server request router | JSON-RPC-like method names to `ClientRequest` variants and domain processors | `MessageProcessor` |
| Serialization router | selected initialized requests into FIFO/shared queues | `RequestSerializationQueues` |
| Domain handlers | thread, turn, fs, command, process, MCP, plugin, account, config behavior | `app-server/src/request_processors/*` |
| Outgoing router | responses, errors, notifications, server-to-client requests to connections | `OutgoingMessageSender` and `route_outgoing_envelope` |
| Core op router | `Op` values to session handlers | `core/src/session/handlers.rs` |
| Exec-server RPC router | remote process/fs/http method names to `ExecServerHandler` methods | `exec-server/src/rpc.rs` and `server/registry.rs` |

The important design lesson: public API routing and core runtime routing are
not the same table. App-server routes client API methods to domain processors.
The core session routes already-normalized `Op` values to agent/runtime
handlers.

## CLI Routing

The first router is the `codex` binary in `codex-rs/cli/src/main.rs`.

The `Subcommand` enum defines the top-level product surfaces. Examples include:

| CLI command | Routed to |
|---|---|
| default `codex` | interactive TUI via `run_interactive_tui` |
| `codex exec` | non-interactive runner via `codex_exec::run_main` |
| `codex review` | builds an `ExecCli` whose command is `ExecCommand::Review`, then calls `codex_exec::run_main` |
| `codex mcp-server` | `codex_mcp_server::run_main` |
| `codex mcp` | MCP configuration CLI |
| `codex plugin` | plugin command handlers such as `run_plugin_add`, `run_plugin_list`, `run_plugin_remove`, marketplace routing |
| `codex app-server` | `codex_app_server::run_main_with_transport_options` |
| `codex app-server daemon ...` | `codex_app_server_daemon` lifecycle commands |
| `codex app-server proxy` | `codex_stdio_to_uds::run` |
| `codex remote-control` | `remote_control_cmd::run` |
| `codex cloud` | `codex_cloud_tasks::run_main` |
| `codex exec-server` | `run_exec_server_command` |

Concrete dispatch examples:

- `Some(Subcommand::Exec(mut exec_cli))` inherits root options, rejects remote
  mode for that surface, then calls `codex_exec::run_main(exec_cli,
  arg0_paths.clone())`.
- `Some(Subcommand::Review(...))` constructs `ExecCli::try_parse_from(["codex",
  "exec"])`, sets `exec_cli.command = Some(ExecCommand::Review(review_args))`,
  then calls `codex_exec::run_main`.
- `Some(Subcommand::AppServer(app_server_cli))` either starts the app-server
  process with `run_main_with_transport_options` or routes to daemon/proxy/schema
  subcommands.
- `run_exec_server_command` chooses between remote environment mode
  (`run_remote_environment`) and local listening mode
  (`run_main_with_telemetry`).

From-scratch implication: build a thin CLI router first. It should parse
commands, normalize common flags, reject unsupported flag combinations, and
delegate to product-specific entry points. Do not place business logic in the
CLI router.

## App-Server Transport Routing

The app-server transport layer converts external messages into
`TransportEvent` values.

`codex-rs/app-server-transport/src/transport/mod.rs` defines:

```rust
pub enum TransportEvent {
    ConnectionOpened { connection_id, origin, writer, disconnect_sender },
    ConnectionClosed { connection_id },
    IncomingMessage { connection_id, message },
}
```

It also defines connection origin:

```rust
pub enum ConnectionOrigin {
    Stdio,
    InProcess,
    WebSocket,
    RemoteControl,
}
```

Supported transport selection is represented by `AppServerTransport`:

- `Stdio`
- `UnixSocket { socket_path }`
- `WebSocket { bind_address }`
- `Off`

The parser accepts `stdio://`, `unix://`, `unix://PATH`, `ws://IP:PORT`, and
`off`.

### Incoming Message Routing

The path is:

1. A transport receives a serialized payload.
2. `forward_incoming_message` parses it as `JSONRPCMessage`.
3. `enqueue_incoming_message` wraps it as `TransportEvent::IncomingMessage`.
4. The event is pushed into the bounded `transport_event_tx` queue.
5. The app-server processor loop consumes the event.

Backpressure is explicit:

- Transport queues use `CHANNEL_CAPACITY = 128`.
- If the incoming queue is full and the message is a request,
  `enqueue_incoming_message` sends a JSON-RPC error with code `-32001` and
  message `Server overloaded; retry later.`
- If the incoming queue is full and the message is a response or other event,
  the transport waits by using `transport_event_tx.send(event).await`.

From-scratch implication: every transport should normalize into the same
connection event type. That keeps stdio, WebSocket, Unix socket, remote-control,
and in-process callers from each inventing their own server behavior.

## App-Server Process Loops

`codex-rs/app-server/src/lib.rs` runs two important app-server tasks.

### Outbound Router Task

The outbound task owns:

```rust
HashMap<ConnectionId, OutboundConnectionState>
```

It receives `OutboundControlEvent` values:

- `Opened`
- `Closed`
- `DisconnectAll`

It also receives outgoing envelopes from `outgoing_rx` and calls
`route_outgoing_envelope(&mut outbound_connections, envelope).await`.

This means request handling does not write directly to sockets. Handlers enqueue
outgoing messages; a dedicated task routes them to the right connection or
broadcasts them.

### Processor Task

The processor task owns:

- `MessageProcessor`
- live connection map: `HashMap<ConnectionId, ConnectionState>`
- connection cleanup tasks
- remote-control status watcher
- running-turn count watcher
- thread-created receiver

It consumes `TransportEvent`:

| Transport event | Processor behavior |
|---|---|
| `ConnectionOpened` | creates outbound state, creates `ConnectionState`, inserts both by `ConnectionId` |
| `ConnectionClosed` | removes connection, closes connection RPC gate, schedules `processor.connection_closed`, removes outbound state |
| `IncomingMessage::Request` | calls `processor.process_request` |
| `IncomingMessage::Response` | calls `processor.process_response` |
| `IncomingMessage::Notification` | calls `processor.process_notification` |
| `IncomingMessage::Error` | calls `processor.process_error` |

After a request transitions a connection from not initialized to initialized,
the processor loop:

1. copies opt-out and experimental state into outbound state,
2. sends initialize notifications to that connection,
3. sends current remote-control status to that connection,
4. calls `processor.connection_initialized`,
5. marks outbound initialized.

From-scratch implication: initialization is not only a normal request. It
changes connection state and unlocks outbound notification routing.

## Request Deserialization

App-server receives a `JSONRPCRequest`, but domain handlers do not process raw
JSON.

`MessageProcessor::process_request` does this:

1. logs method and request id,
2. builds `ConnectionRequestId { connection_id, request_id }`,
3. creates a tracing `RequestContext`,
4. registers that context,
5. calls `deserialize_client_request(&request)`,
6. calls `handle_client_request` with the typed `ClientRequest`,
7. sends an error through `OutgoingMessageSender` if deserialization or handling
   fails.

`deserialize_client_request` is intentionally simple:

```rust
serde_json::to_value(request)
    .and_then(serde_json::from_value::<ClientRequest>)
```

The method string is therefore decoded by serde into the generated
`ClientRequest` enum in `codex-rs/app-server-protocol/src/protocol/common.rs`.

The app-server envelope is JSON-RPC-like, but the protocol file notes elsewhere
that it is not strict JSON-RPC 2.0 because the wire format does not send or
expect `"jsonrpc": "2.0"`.

## Typed Method Routing

`codex-rs/app-server-protocol/src/protocol/common.rs` uses the
`client_request_definitions!` macro to generate:

- `enum ClientRequest`
- `ClientRequest::id()`
- `ClientRequest::method()`
- `ClientRequest::serialization_scope()`
- typed `ClientResponse`
- response export support

Each request variant has:

- a wire method string such as `thread/start` or `turn/start`,
- a params type,
- a response type,
- a serialization rule.

The protocol layer owns the request shape. The app-server layer owns the
behavior.

## Initialization Gate

`MessageProcessor::handle_client_request` has a two-stage routing model:

1. `initialize` is allowed before the connection is initialized.
2. Initialized-only requests are rejected with `invalid_request("Not initialized")`
   until the session is initialized.

Before an initialized request is dispatched, app-server also checks:

- experimental API gating via `codex_request.experimental_reason()`,
- connection client name/version,
- OpenAI form elicitation support,
- request serialization scope.

From-scratch implication: do not let every handler check initialization. Put
the connection readiness gate in the central dispatcher.

## Request Serialization Routing

Routing is not only about handler selection. Some requests must not run in
parallel with other requests that touch the same resource.

`ClientRequest::serialization_scope()` returns an optional
`ClientRequestSerializationScope` from `app-server-protocol/src/protocol/common.rs`.

Available scopes:

```rust
pub enum ClientRequestSerializationScope {
    Global(&'static str),
    GlobalSharedRead(&'static str),
    Thread { thread_id: String },
    ThreadPath { path: PathBuf },
    CommandExecProcess { process_id: String },
    Process { process_handle: String },
    FuzzyFileSearchSession { session_id: String },
    FsWatch { watch_id: String },
    McpOauth { server_name: String },
}
```

`MessageProcessor` maps the protocol scope into a queue key by calling:

```rust
RequestSerializationQueueKey::from_scope(connection_id, scope)
```

`RequestSerializationQueueKey` adds connection id where needed:

- command exec process: `(connection_id, process_id)`
- process session: `(connection_id, process_handle)`
- filesystem watch: `(connection_id, watch_id)`

This matters because two clients may reuse the same process id or watch id
string. Connection-local resources should not collide globally.

### Exclusive vs Shared Read

`RequestSerializationAccess` has two modes:

- `Exclusive`
- `SharedRead`

The queue drain algorithm:

1. pops the first request for a key,
2. if it is `SharedRead`, also drains consecutive queued `SharedRead` requests,
3. runs that batch with `join_all`,
4. returns to the queue,
5. removes the queue when empty.

So reads for the same global key can run together, but writes serialize with
reads and other writes.

### Concrete Scope Examples

The tests in `common.rs` document representative scopes:

| Request | Serialization scope |
|---|---|
| `thread/resume` with thread id | `Thread { thread_id }` |
| `thread/fork` | `Thread { thread_id }` |
| `command/exec` with `process_id` | `CommandExecProcess { connection_id, process_id }` |
| `fuzzyFileSearch/sessionUpdate` | `FuzzyFileSearchSession { session_id }` |
| `fs/watch` | `FsWatch { connection_id, watch_id }` |
| `plugin/install` | `Global("config")` |
| `skills/list` | `GlobalSharedRead("config")` |
| `skills/extraRoots/set` | `Global("config")` |
| `plugin/uninstall` | `Global("config")` |
| `mcpServer/oauth/login` | `McpOauth { server_name }` |
| `mcpServer/resource/read` with thread id | `Thread { thread_id }` |
| `config/read` | `GlobalSharedRead("config")` |
| `account/get` | `Global("account-auth")` |
| `thread/goal/set` | `Thread { thread_id }` |
| `environment/add` | `Global("environment")` |
| `remoteControl/pairing/start` | `Global("remote-control-pairing")` |
| `remoteControl/pairing/status` | `GlobalSharedRead("remote-control-pairing")` |
| `remoteControl/clients/list` | `GlobalSharedRead("remote-control-clients")` |
| `remoteControl/clients/revoke` | `Global("remote-control-clients")` |

Requests such as `thread/start`, `fs/readFile`, `thread/turns/list`, and
`thread/items/list` have no serialization scope in the representative tests.

From-scratch implication: define concurrency/resource scopes in the protocol
metadata, not scattered across handlers. Then the dispatcher can enforce them
before handler code runs.

## App-Server Domain Dispatch Table

`MessageProcessor::handle_initialized_client_request` is the central app-server
dispatch table.

Its pattern is:

```rust
match codex_request {
    ClientRequest::SomeMethod { params, .. } => {
        self.some_processor.some_method(request_id, params).await
    }
}
```

The return type is:

```rust
Result<Option<ClientResponsePayload>, JSONRPCErrorError>
```

That creates three handler completion shapes:

| Handler result | Meaning |
|---|---|
| `Ok(Some(response))` | central dispatcher sends a JSON-RPC response immediately |
| `Ok(None)` | handler already sent a response, will send one later, or uses notifications |
| `Err(error)` | central dispatcher sends a JSON-RPC error |

After the match:

```rust
Ok(Some(response)) => outgoing.send_response_as(...)
Ok(None) => {}
Err(error) => outgoing.send_error(...)
```

### Domain Processor Ownership

Representative app-server request ownership:

| Request family | Handler owner |
|---|---|
| `config/read`, `config/value/write`, config requirements, model provider capabilities | `config_processor` |
| Windows sandbox setup/readiness | `windows_sandbox_processor` |
| external agent config import/detect | `external_agent_config_processor` |
| remote-control enable/disable/status/pairing/clients | `remote_control_processor` |
| environment add/info | `environment_processor` |
| `fs/*` | `fs_processor` |
| `thread/start`, `thread/resume`, `thread/fork`, archive/delete/name/list/read/items | `thread_processor` |
| thread goals | `thread_goal_processor` |
| `turn/start`, realtime, interrupt, review, thread settings | `turn_processor` |
| skills, hooks, models, permissions, collaboration modes | `catalog_processor` |
| marketplace | `marketplace_processor` |
| plugins | `plugin_processor` |
| apps | `apps_processor` |
| MCP OAuth/status/resource/tool | `mcp_processor` |
| account login/logout/status/usage/workspace messages | `account_processor` |
| git diff | `git_processor` |
| fuzzy file search | `search_processor` |
| one-off command exec | `command_exec_processor` |
| connection-scoped process exec | `process_exec_processor` |
| feedback upload | `feedback_processor` |

From-scratch implication: keep a central dispatcher, but make handler ownership
domain-oriented. The dispatcher should be boring; complexity belongs in domain
processors.

## Direct Response, Late Response, and Notification Routing

App-server responses are routed through `OutgoingMessageSender` in
`codex-rs/app-server/src/outgoing_message.rs`.

Important outgoing types:

```rust
pub(crate) enum OutgoingEnvelope {
    ToConnection { connection_id, message, write_complete_tx },
    Broadcast { message },
}
```

`OutgoingMessageSender` owns:

- `next_server_request_id`
- outgoing envelope channel
- server-request callbacks by `RequestId`
- request contexts by `ConnectionRequestId`
- analytics client

### Direct Responses

`send_response` and `send_response_as`:

1. convert a typed `ClientResponsePayload` into JSON-RPC response parts,
2. take and remove the registered request context,
3. enqueue `OutgoingEnvelope::ToConnection`,
4. target the original `connection_id`.

### Direct Errors

`send_error`:

1. takes and removes the registered request context,
2. builds `OutgoingMessage::Error`,
3. enqueues it to the original connection.

### Broadcast Notifications

`send_server_notification` calls:

```rust
send_server_notification_to_connections(&[], notification)
```

An empty connection list means broadcast. The outbound router handles the actual
fanout.

### Targeted Notifications

`send_server_notification_to_connections` with explicit connection ids enqueues
one `OutgoingEnvelope::ToConnection` per target.

`send_server_notification_to_connection_and_wait` includes a oneshot
`write_complete_tx` and waits until the outbound writer has completed the write.
This is used when ordering or delivery acknowledgement matters for a local
operation.

### Thread-Originated Responses

`send_response_with_thread_originator` is used by thread lifecycle paths to
attach originator context to analytics. It still routes to the original request
connection.

From-scratch implication: handlers should never know how to write to a socket.
They should publish typed outgoing work to an outbound message bus.

## Server-to-Client Request Routing

Some app-server operations need the client to answer a question, such as
approval, tool input, authentication token refresh, or current time.

`OutgoingMessageSender` supports this with pending callbacks:

- generates server request ids,
- sends `ServerRequest` messages to one or more connections,
- stores a callback in `request_id_to_callback`,
- resolves the callback when the matching client response arrives,
- aborts pending callbacks on thread cleanup or connection cleanup.

`ThreadScopedOutgoingMessageSender` narrows delivery to connections associated
with a thread and records the thread id for cleanup.

From-scratch implication: bidirectional RPC needs its own correlation table. Do
not confuse client request ids with server request ids.

## Connection Cleanup Routing

Connection cleanup is part of routing correctness.

On `TransportEvent::ConnectionClosed`, the processor loop:

1. removes the connection from the live connection map,
2. closes `connection_state.session.rpc_gate`,
3. removes outbound state,
4. spawns `processor.connection_closed(connection_id, &connection_state.session)`.

The RPC gate prevents new gated work from continuing after the connection is
closed. Domain cleanup then releases connection-scoped resources such as process
sessions, command streams, fs watches, and pending requests.

From-scratch implication: any resource keyed by connection id needs a cleanup
path triggered by connection close, not only by explicit `kill` or `unwatch`
requests.

## In-Process App-Server Routing

`codex-rs/app-server/src/in_process.rs` runs app-server without stdio or socket
I/O.

The module-level docs state the intent:

- run the existing `MessageProcessor` and outbound routing logic on Tokio tasks,
- replace socket/stdio transports with bounded in-memory channels,
- preserve app-server semantics,
- use typed `ClientRequest` values on the incoming path,
- keep responses in the same JSON-RPC result envelope.

In-process client messages include:

```rust
enum InProcessClientMessage {
    Request { request, response_tx },
    Notification { notification },
    ServerRequestResponse { request_id, result },
    ServerRequestError { request_id, error },
    Shutdown { done_tx },
}
```

Typed requests are handled by:

```rust
MessageProcessor::process_client_request
```

This bypasses JSON deserialization but calls the same `handle_client_request`
used by external transports. In-process callers therefore share handler
semantics, initialization behavior, serialization scopes, and outgoing routing.

Backpressure:

- command submission uses `try_send`,
- saturated request queues return `WouldBlock`,
- notification fanout may drop events under saturation,
- server requests are failed back into `MessageProcessor` rather than silently
  abandoned.

From-scratch implication: if you need embedded and remote clients, avoid making
two server implementations. Use one processor and two transport adapters.

## Core Operation Routing

After app-server handlers normalize client requests, many actions enter the
core session as `Op` values from `codex-rs/protocol/src/protocol.rs`.

The core operation router is `submission_loop` in
`codex-rs/core/src/session/handlers.rs`.

It receives `Submission` values and matches `sub.op`:

| `Op` variant | Handler |
|---|---|
| `Interrupt` | `interrupt(&sess)` |
| `CleanBackgroundTerminals` | `clean_background_terminals(&sess)` |
| realtime start/audio/text/speech/close/list voices | realtime handlers |
| `UserInput { .. }` | `user_input_or_turn(&sess, sub.id, sub.op, sub.client_user_message_id)` |
| `ThreadSettings` | `update_thread_settings` |
| `InterAgentCommunication` | `inter_agent_communication` |
| `ExecApproval` | `exec_approval` |
| `PatchApproval` | `patch_approval` |
| `UserInputAnswer` | `request_user_input_response` |
| `RequestPermissionsResponse` | `request_permissions_response` |
| `DynamicToolResponse` | `dynamic_tool_response` |
| `RefreshMcpServers` | `refresh_mcp_servers` |
| `ReloadUserConfig` | `reload_user_config` |
| `Compact` | `compact` |
| `ThreadRollback` | `thread_rollback` |
| `SetThreadMemoryMode` | `set_thread_memory_mode` |
| `RunUserShellCommand` | `run_user_shell_command` |
| `ResolveElicitation` | `resolve_elicitation` |
| `Shutdown` | `shutdown` |
| `Review` | `review` |
| `ApproveGuardianDeniedAction` | `approve_guardian_denied_action` |

The enum is non-exhaustive, and unknown operations are ignored by the wildcard
arm.

This router is not exposed as app-server API. It is the internal session
command bus.

## Core Task Routing

`codex-rs/core/src/tasks/mod.rs` defines `SessionTask`.

Important mechanics:

- `SessionTask` implementations expose `kind`, `span_name`, `run`, and `abort`.
- `Session::spawn_task` aborts previous task state and starts a new task.
- `Session::start_task` uses a cancellation token, active turn tracking,
  tracing span, and `tokio::spawn`.
- On task finish, it flushes rollout data and calls `on_task_finished`.

From-scratch implication: session operation dispatch and long-running task
execution should be separate. The op router decides what to do. The task layer
owns cancellation, active-turn state, and cleanup.

## Exec-Server RPC Routing

The exec-server has its own JSON-RPC router, separate from app-server.

`codex-rs/exec-server/src/rpc.rs` defines:

```rust
pub(crate) struct RpcRouter<S> {
    request_routes: HashMap<&'static str, RequestRoute<S>>,
    notification_routes: HashMap<&'static str, NotificationRoute<S>>,
}
```

It has three registration styles:

| Registration method | Behavior |
|---|---|
| `request<P, R, F, Fut>` | decode params, await handler, serialize response or error |
| `request_with_id<P, F, Fut>` | decode params, pass request id into handler, handler may send later output, no immediate response on success |
| `notification<P, F, Fut>` | decode notification params, await handler, no response |

`codex-rs/exec-server/src/server/registry.rs` registers routes:

| Method | Handler |
|---|---|
| `initialized` notification | `handler.initialized()` |
| `initialize` | `handler.initialize(params)` |
| `http/request` | `handler.http_request(request_id, params)` via `request_with_id` |
| `process/start` | `handler.exec(params)` |
| `environment/info` | `handler.environment_info()` |
| `process/read` | `handler.exec_read(params)` |
| `process/write` | `handler.exec_write(params)` |
| `process/signal` | `handler.signal(params)` |
| `process/terminate` | `handler.terminate(params)` |
| `fs/readFile` | `handler.fs_read_file(params)` |
| `fs/open` | `handler.fs_open(params)` |
| `fs/readBlock` | `handler.fs_read_block(params)` |
| `fs/close` | `handler.fs_close(params)` |
| `fs/writeFile` | `handler.fs_write_file(params)` |
| `fs/createDirectory` | `handler.fs_create_directory(params)` |
| `fs/getMetadata` | `handler.fs_get_metadata(params)` |
| `fs/canonicalize` | `handler.fs_canonicalize(params)` |
| `fs/readDirectory` | `handler.fs_read_directory(params)` |
| `fs/walk` | `handler.fs_walk(params)` |
| `fs/remove` | `handler.fs_remove(params)` |
| `fs/copy` | `handler.fs_copy(params)` |

From-scratch implication: app-server and exec-server should have different
protocols. App-server routes product and conversation APIs. Exec-server routes
remote filesystem/process primitives.

## Error Routing

There are multiple error layers:

| Error point | Route |
|---|---|
| invalid JSON-RPC message at transport parse | log and keep connection alive |
| overloaded incoming request queue | immediate JSON-RPC error `-32001` |
| unknown or invalid app-server request payload | `deserialize_client_request` returns `invalid_request` |
| request before initialization | central handler returns `invalid_request("Not initialized")` |
| experimental API used without opt-in | central handler returns `invalid_request(...)` |
| handler validation failure | handler returns `JSONRPCErrorError` |
| response serialization failure | `send_response_as_inner` sends internal error |
| connection closed while request pending | connection cleanup/gate aborts pending work |
| exec-server param decode failure | `RpcRouter` returns JSON-RPC error |
| exec-server handler failure | `RpcRouter` serializes handler error |

The pattern is consistent: errors are routed to the original request connection
when there is a request id. Notifications and cleanup errors are logged or
surfaced as notifications depending on domain behavior.

## Request Correlation Keys

The system uses several distinct ids. They must not be collapsed.

| Id | Purpose |
|---|---|
| `ConnectionId` | identifies the client connection |
| `RequestId` | JSON-RPC-like request id from client or generated for server-to-client requests |
| `ConnectionRequestId` | pair of `ConnectionId` and `RequestId`; routes responses to the right connection |
| thread id | conversation/thread resource id |
| turn id | core submission id for a turn |
| process id | one-off command process id, optionally client supplied |
| process handle | connection-scoped long-running process handle |
| fs watch id | connection-scoped filesystem watcher id |
| MCP server name | key for MCP OAuth serialization |

From-scratch implication: use compound keys where resource ids are
connection-local. A bare `process_id` or `watch_id` is not enough.

## Request-to-Handler Examples

### `thread/start`

| Step | Concrete route |
|---|---|
| Wire method | `thread/start` |
| Typed request | `ClientRequest::ThreadStart` |
| Serialization scope | none |
| Dispatcher | `MessageProcessor::handle_initialized_client_request` |
| Handler | `thread_processor.thread_start(...)` |
| Completion shape | returns `Ok(None)`; background task sends response and notification |
| Outgoing | `send_response_with_thread_originator`, then `ThreadStarted` notification |

### `turn/start`

| Step | Concrete route |
|---|---|
| Wire method | `turn/start` |
| Typed request | `ClientRequest::TurnStart` |
| Serialization scope | `thread_id(params.thread_id)` in `common.rs`, mapped to `Thread { thread_id }` |
| Dispatcher | `MessageProcessor::handle_initialized_client_request` |
| Handler | `turn_processor.turn_start(...)` |
| Core route | submits `Op::UserInput` |
| Core dispatch | `submission_loop` routes `Op::UserInput` to `user_input_or_turn` |
| Completion shape | immediate `TurnStartResponse`, later notifications for events/items/completion |

### `fs/readFile`

| Step | Concrete route |
|---|---|
| Wire method | `fs/readFile` |
| Typed request | `ClientRequest::FsReadFile` |
| Serialization scope | none in representative tests |
| Handler | `fs_processor.read_file(params)` |
| Completion shape | `Ok(Some(FsReadFileResponse))` |
| Payload detail | file bytes returned base64-encoded at the API boundary |

### `command/exec`

| Step | Concrete route |
|---|---|
| Wire method | `command/exec` |
| Typed request | `ClientRequest::OneOffCommandExec` |
| Serialization scope | `CommandExecProcess { connection_id, process_id }` when `process_id` is present |
| Handler | `command_exec_processor.one_off_command_exec(&request_id, params)` |
| Completion shape | often `Ok(None)`; command manager sends response/notifications |
| Cleanup | connection-scoped command state is cleaned on connection close |

### `process/spawn`

| Step | Concrete route |
|---|---|
| Wire method | `process/spawn` |
| Typed request | `ClientRequest::ProcessSpawn` |
| Serialization scope | keyed by process handle for subsequent operations |
| Handler | `process_exec_processor.process_spawn(request_id, params)` |
| Completion shape | maps to `Ok(None)` after spawn path starts; spawn response can be sent from manager |
| Cleanup | connection-scoped process sessions |

### `mcpServer/oauth/login`

| Step | Concrete route |
|---|---|
| Wire method | `mcpServer/oauth/login` |
| Typed request | `ClientRequest::McpServerOauthLogin` |
| Serialization scope | `McpOauth { server_name }` |
| Handler | `mcp_processor.mcp_server_oauth_login(params)` |
| Completion shape | async/domain-specific response and notifications |

### Exec-server `process/start`

| Step | Concrete route |
|---|---|
| Wire method | `process/start` |
| Router | `RpcRouter<ExecServerHandler>` |
| Registration | `router.request(EXEC_METHOD, ...)` |
| Handler | `ExecServerHandler::exec(params)` |
| Response | serialized handler result or JSON-RPC error |

## How To Implement This Routing Layer From Scratch

Build it in this order:

1. Define one external request envelope with `id`, `method`, `params`, and
   optional trace metadata.
2. Define one internal typed request enum generated or hand-written from method
   strings.
3. Put params and response types next to protocol definitions.
4. Create a transport-neutral event enum: connection opened, connection closed,
   incoming message.
5. Give every connection a stable `ConnectionId`.
6. Add a central processor loop that owns the connection map.
7. Add an initialization gate before normal request dispatch.
8. Add protocol metadata for request serialization scopes.
9. Implement keyed FIFO queues with exclusive and shared-read access.
10. Add a central dispatcher from typed request variants to domain processors.
11. Make handlers return one of three shapes: direct response, no immediate
    response, or error.
12. Create an outgoing envelope bus with targeted and broadcast messages.
13. Store request contexts by `(connection_id, request_id)` until response or
    error.
14. Add server-to-client request ids and callback storage separately from
    client request ids.
15. Add connection cleanup that closes gates and releases connection-scoped
    resources.
16. Implement in-process transport by reusing the same processor, not by
    duplicating handlers.
17. Define an internal core operation enum such as `Op`.
18. Route core operations separately from public API methods.
19. Add a task layer for long-running work, cancellation, and active-turn state.
20. If remote execution is needed, define a separate exec-server protocol and
    router for filesystem/process primitives.

## Non-Handwavy Invariants

- A client request is never handled by a domain processor until it has become a
  typed `ClientRequest`.
- Initialization is a connection-level gate.
- Outbound responses are routed by `ConnectionRequestId`, not only by
  `RequestId`.
- Notifications can be broadcast or targeted.
- Server-to-client requests have their own generated request ids and callback
  table.
- Requests with serialization scopes are gated before domain dispatch.
- Shared-read scopes may run together only when they are consecutive in the
  same queue.
- Connection-local resources include connection id in their queue key.
- App-server handlers do not write directly to transport sockets.
- In-process app-server clients bypass JSON parsing but reuse the same
  `handle_client_request` path.
- Core session routing uses `Op`, not app-server method strings.
- Exec-server has a separate router and protocol from app-server.
