# Layer 12 - Design Patterns

This layer explains the high-level and low-level design patterns used by the
Codex codebase. It is not a generic patterns catalog. Each pattern below is
anchored to concrete source files and explains what you would implement if you
were building this system from scratch.

## Source Anchors

Primary files read for this layer:

| Area | Source |
|---|---|
| CLI command router | `codex-rs/cli/src/main.rs` |
| App-server typed protocol | `codex-rs/app-server-protocol/src/protocol/common.rs` |
| App-server request handler | `codex-rs/app-server/src/message_processor.rs` |
| App-server transport loops | `codex-rs/app-server/src/lib.rs` |
| Request serialization queues | `codex-rs/app-server/src/request_serialization.rs` |
| Outgoing response/notification router | `codex-rs/app-server/src/outgoing_message.rs` |
| In-process app-server transport | `codex-rs/app-server/src/in_process.rs` |
| Core thread manager | `codex-rs/core/src/thread_manager.rs` |
| Core thread facade | `codex-rs/core/src/codex_thread.rs` |
| Core operation/event protocol | `codex-rs/protocol/src/protocol.rs` |
| Core operation dispatch | `codex-rs/core/src/session/handlers.rs` |
| Exec-server RPC router | `codex-rs/exec-server/src/rpc.rs` |
| Storage port | `codex-rs/thread-store/src/store.rs` |
| Append-only rollout writer | `codex-rs/rollout/src/recorder.rs` |

## Pattern Map

| Pattern | Codex example | Design job |
|---|---|---|
| Thin entrypoint router | `Subcommand` dispatch in `cli/src/main.rs` | Keep process startup separate from product logic. |
| Typed protocol boundary | `ClientRequest` macro in `app-server-protocol/.../common.rs` | Convert wire methods into typed request variants. |
| JSON envelope, typed core | `deserialize_client_request` and `handle_client_request` | Accept JSON at the edge, avoid JSON inside handlers. |
| Initialization gate | `Initialize` special-case in `MessageProcessor` | Prevent normal methods before connection setup. |
| Capability/feature gate | `experimental_reason()` check in `MessageProcessor` | Block unsupported methods/fields by connection capability. |
| Resource-scoped serialization | `RequestSerializationQueues` | Preserve ordering only where shared resources require it. |
| Domain processor ownership | `config_processor`, `thread_processor`, `fs_processor` dispatch | Keep handler logic grouped by resource. |
| Response plus event stream | `OutgoingMessageSender` and server notifications | Separate request acknowledgement from ongoing progress. |
| Thread/session facade | `CodexThread::submit(Op)` | Give callers one stable way to drive a conversation. |
| Command/event split | `Op` and `EventMsg` | Send commands in, emit facts out. |
| Composition root | `ThreadManagerState` | Assemble services once, inject them into threads. |
| Port/adapter storage | `ThreadStore` trait | Decouple core from local/in-memory storage implementation. |
| Append-only history plus projection | `RolloutRecorder` plus thread metadata/index | Preserve replayable history while serving fast UI reads. |
| Background worker | app-server loops, rollout writer, core submission loop | Move long-running work into owned async tasks. |
| Router registry | `RpcRouter` in exec-server | Register method names to typed handlers. |
| Transport substitution | `in_process.rs` | Reuse protocol semantics without requiring sockets. |

## High-Level Patterns

### 1. Thin Entrypoint Router

**Problem solved:** A single binary exposes many product surfaces: interactive
TUI, non-interactive exec, review, app-server, app-server daemon, MCP, plugin
commands, cloud commands, and exec-server. If all behavior lived in the binary
entrypoint, the system would become impossible to reason about.

**Where it appears:**

- `codex-rs/cli/src/main.rs` defines `Subcommand`.
- The dispatch match around `Subcommand::Exec`, `Subcommand::Review`,
  `Subcommand::AppServer`, and `Subcommand::ExecServer` calls crate-level
  runners such as `codex_exec::run_main(...)` and
  `codex_app_server::run_main_with_transport_options(...)`.

**How Codex uses it:**

The CLI parses arguments and applies surface-specific startup policy, then
hands control to the crate that owns the real behavior. For example, `review`
is normalized into an `ExecCli` with `ExecCommand::Review`, while `app-server`
constructs transport/runtime options before calling app-server code.

**From-scratch implementation rule:**

Start with a thin `main` that does only four jobs:

1. Parse command-line args.
2. Load config overrides.
3. Select a product surface.
4. Call the owning module's `run_*` function.

Do not put agent loop logic, storage logic, or API handler logic in `main`.

### 2. Typed Protocol Boundary

**Problem solved:** Rich clients send method strings and JSON params. The
server needs handler code to work with typed data, not raw JSON blobs.

**Where it appears:**

- `codex-rs/app-server-protocol/src/protocol/common.rs` defines
  `ClientRequestSerializationScope`.
- The `client_request_definitions!` macro generates the `ClientRequest` enum,
  method IDs, response payload conversions, and serialization-scope behavior.
- `TurnStart => "turn/start"` maps to `v2::TurnStartParams` and
  `v2::TurnStartResponse`.

**How Codex uses it:**

Each app-server request has:

- a wire method string such as `turn/start`;
- a params struct such as `TurnStartParams`;
- a response struct such as `TurnStartResponse`;
- a `ClientRequest` enum variant;
- optional serialization-scope metadata.

**From-scratch implementation rule:**

Define the protocol as data first. For each method, specify:

| Requirement | Example |
|---|---|
| method string | `turn/start` |
| request params | `TurnStartParams` |
| response payload | `TurnStartResponse` |
| notification payloads | `turn/started`, `item/started`, `turn/completed` |
| ordering key | thread ID |
| handler owner | `TurnProcessor` |

Then generate or manually maintain one typed enum that represents every
client request.

### 3. JSON Envelope, Typed Internal Request

**Problem solved:** JSON-RPC-like transports are flexible, but application
logic should not be littered with ad hoc JSON extraction.

**Where it appears:**

- `codex-rs/app-server/src/message_processor.rs`
  - `process_request(...)`
  - `deserialize_client_request(...)`
  - `handle_client_request(...)`
  - `process_client_request(...)`

**How Codex uses it:**

Socket/stdio requests arrive as `JSONRPCRequest`. `process_request` registers
request context, deserializes into `ClientRequest`, and delegates to
`handle_client_request`. In-process embedders skip JSON deserialization but
still call the same typed request path through `process_client_request`.

This gives two entry paths with one behavior contract:

```text
JSON transport -> JSONRPCRequest -> ClientRequest -> handler
in-process API  -> ClientRequest                 -> handler
```

**From-scratch implementation rule:**

Use raw JSON only at the transport edge. Immediately convert to a typed request
enum. Every domain handler should accept typed params.

### 4. Initialization Gate

**Problem solved:** Some connection state must exist before normal request
handling: client capabilities, experimental API settings, notification
preferences, auth/session fields, and outbound readiness.

**Where it appears:**

- `MessageProcessor::handle_client_request(...)` special-cases
  `ClientRequest::Initialize`.
- `dispatch_initialized_client_request(...)` rejects non-initialize requests
  when `!session.initialized()`.

**How Codex uses it:**

`Initialize` is the only request allowed before a connection is initialized.
After it succeeds, app-server sends initialize notifications and marks the
connection ready for outbound messages.

**From-scratch implementation rule:**

Treat connection initialization as a state transition:

```text
NewConnection -> InitializeReceived -> Initialized -> NormalRequestsAllowed
```

Do not allow `thread/start`, `turn/start`, config writes, filesystem writes, or
tool approvals before this transition.

### 5. Capability and Experimental API Gate

**Problem solved:** Different clients can support different features. Some API
surface is experimental and must be opt-in.

**Where it appears:**

- `MessageProcessor::dispatch_initialized_client_request(...)` checks
  `codex_request.experimental_reason()` and `session.experimental_api_enabled()`.
- `ConnectionCapabilities` is passed to thread connection setup.

**How Codex uses it:**

The typed request knows whether it requires an experimental feature. The
connection session knows whether that feature class is enabled. The dispatcher
enforces the gate before the request reaches domain handlers.

**From-scratch implementation rule:**

Put capability checks in the request dispatcher, not inside every handler.
Handlers should generally receive only requests they are allowed to execute.

### 6. Resource-Scoped Serialization Queue

**Problem solved:** The server handles many concurrent requests, but some
requests must not race: two writes to the same thread, two config mutations,
two operations on the same process handle, or an OAuth flow for the same MCP
server.

**Where it appears:**

- `codex-rs/app-server-protocol/src/protocol/common.rs`
  - `ClientRequestSerializationScope`
  - request-specific `serialization_scope()`
- `codex-rs/app-server/src/request_serialization.rs`
  - `RequestSerializationQueueKey`
  - `RequestSerializationAccess`
  - `RequestSerializationQueues::enqueue(...)`
  - `RequestSerializationQueues::drain(...)`

**How Codex uses it:**

Requests declare their serialization scope at the protocol layer. The app
server converts that scope into an internal queue key. Requests with the same
key are drained in FIFO order. Consecutive `SharedRead` requests can run
together, while exclusive requests block other requests on the same key.

Examples of queue keys:

- `Global("config")`
- `Thread { thread_id }`
- `ThreadPath { path }`
- `CommandExecProcess { connection_id, process_id }`
- `Process { connection_id, process_handle }`
- `FsWatch { connection_id, watch_id }`
- `McpOauth { server_name }`

**From-scratch implementation rule:**

Do not make the whole server single-threaded to avoid races. Instead:

1. Identify shared resources.
2. Assign each request an optional resource key.
3. Serialize only requests with the same key.
4. Let unrelated requests run concurrently.

### 7. Domain Processor Ownership

**Problem solved:** A single app-server dispatcher owns many APIs. Handler
logic needs stable ownership boundaries so filesystem handling, thread
handling, config handling, account handling, and plugin handling do not merge
into one giant module.

**Where it appears:**

- `MessageProcessor::handle_initialized_client_request(...)` matches
  `ClientRequest` variants and delegates to processors:
  - `config_processor`
  - `thread_processor`
  - `fs_processor`
  - `command_exec_processor`
  - `process_exec_processor`
  - `environment_processor`
  - `remote_control_processor`
  - `account_processor`
  - `plugin_processor`

**How Codex uses it:**

`MessageProcessor` is the application-level router. It does not implement
every behavior directly. It maps typed request variants to the processor that
owns the resource.

**From-scratch implementation rule:**

Create one processor per resource family:

| Resource family | Processor |
|---|---|
| threads and turns | `ThreadProcessor`, `TurnProcessor` |
| config | `ConfigProcessor` |
| files | `FsProcessor` |
| commands/processes | `CommandExecProcessor`, `ProcessExecProcessor` |
| auth/account | `AccountProcessor` |
| plugins/skills/apps | dedicated extension processors |

The router should decide where a request goes. The processor should decide how
to execute it.

### 8. Response Plus Notification Stream

**Problem solved:** Some requests finish quickly, but agent turns are long
running. A client needs immediate acknowledgement and then a stream of progress
events.

**Where it appears:**

- `codex-rs/app-server/src/outgoing_message.rs`
  - `OutgoingEnvelope`
  - `OutgoingMessageSender`
  - `send_response(...)`
  - `send_error(...)`
  - `send_server_notification(...)`
  - `send_request_to_connections(...)`
- `codex-rs/app-server/src/message_processor.rs` sends either:
  - direct response payloads;
  - direct errors;
  - no direct response when work completes later through notifications.

**How Codex uses it:**

The app-server path distinguishes:

- request response: correlated to one request ID;
- request error: correlated to one request ID;
- server notification: pushed to one or more connections;
- server request: server asks the client for approval/input and tracks a
  pending callback.

For a turn, `turn/start` can return a `TurnStartResponse`, while progress
continues through notifications such as turn started, item started, item
completed, and turn completed.

**From-scratch implementation rule:**

Do not block the original request until an AI turn is fully complete. Return a
small response with IDs, then stream progress events. The UI should render from
notifications and use read APIs to recover state after reconnect.

### 9. Thread/Session Facade

**Problem solved:** Many surfaces need to interact with a conversation, but
they should not know how the core session loop is implemented.

**Where it appears:**

- `codex-rs/core/src/codex_thread.rs`
  - `CodexThread`
  - `CodexThread::submit(&self, op: Op)`
  - `shutdown_and_wait(...)`
  - `wait_until_terminated(...)`

**How Codex uses it:**

`CodexThread` is the object app-server and other callers use to drive a
conversation. It wraps the internal `Codex` runtime and exposes a small thread
interface.

**From-scratch implementation rule:**

Expose a narrow conversation handle:

```text
ThreadHandle.submit(command)
ThreadHandle.shutdown()
ThreadHandle.wait_until_terminated()
ThreadHandle.subscribe_events()
```

Keep model calls, tool execution, history mutation, and background task
management behind that handle.

### 10. Command/Event Split

**Problem solved:** The system needs a clear distinction between requests to
do work and facts produced by work.

**Where it appears:**

- `codex-rs/protocol/src/protocol.rs`
  - `Op` enum: commands submitted into a session.
  - `EventMsg` enum: events emitted out of a session.
- `codex-rs/core/src/session/handlers.rs`
  - `submission_loop(...)` matches on `Op`.

**How Codex uses it:**

External APIs are translated into `Op` values. The core session loop receives
submissions and dispatches by `Op`. The session emits `EventMsg` values such as
errors, warnings, turn lifecycle events, token counts, agent messages, user
messages, reasoning, tool calls, and completion events.

**From-scratch implementation rule:**

Use this mental model:

```text
Request -> typed params -> Op -> session work -> EventMsg -> storage/UI projection
```

Do not let the UI directly mutate session internals. Do not let storage decide
what the agent should do. Commands enter through one side; events leave through
the other.

### 11. Composition Root / Service Container

**Problem solved:** A thread needs many services: auth, models, environments,
skills, plugins, MCP, storage, telemetry, extension registry, time provider,
and optional stores. These should be assembled once instead of being recreated
inside handlers.

**Where it appears:**

- `codex-rs/core/src/thread_manager.rs`
  - `ThreadManager`
  - `StartThreadOptions`
  - `ThreadManagerState`
  - `thread_store_from_config(...)`

**How Codex uses it:**

`ThreadManagerState` is shared through `Arc`. It stores long-lived services and
the in-memory map of active `ThreadId -> CodexThread`. Thread creation receives
`StartThreadOptions`, then the manager injects already-constructed services
into the thread/session runtime.

**From-scratch implementation rule:**

Build a composition root that owns:

- active thread map;
- auth manager;
- model provider/manager;
- environment manager;
- plugin/skill/tool registries;
- storage implementation;
- telemetry/logging;
- time provider;
- config snapshot.

Pass dependencies into threads instead of constructing them deep in handlers.

### 12. Port and Adapter

**Problem solved:** Core logic should not depend on one concrete storage
backend or execution environment.

**Where it appears:**

- `codex-rs/thread-store/src/store.rs`
  - `ThreadStore` trait
  - methods such as `create_thread`, `resume_thread`, `append_items`,
    `flush_thread`, `read_thread`, `list_threads`, `update_thread_metadata`,
    `archive_thread`, `delete_thread`
- `codex-rs/core/src/thread_manager.rs`
  - `thread_store_from_config(...)` selects `LocalThreadStore` or
    `InMemoryThreadStore`.

**How Codex uses it:**

Core code depends on `Arc<dyn ThreadStore>`. The concrete store can be local
filesystem/SQLite or in-memory. The trait is the port; concrete stores are
adapters.

**From-scratch implementation rule:**

Define trait boundaries for:

- thread storage;
- model provider;
- filesystem/process execution;
- auth storage;
- tool/MCP execution;
- telemetry/logging.

Start with in-memory adapters for tests and local filesystem adapters for real
use.

### 13. Append-Only History Plus Metadata Projection

**Problem solved:** The system needs durable replay history and fast UI list/read
queries. One storage shape rarely serves both perfectly.

**Where it appears:**

- `codex-rs/rollout/src/recorder.rs`
  - `RolloutRecorder`
  - `RolloutCmd::AddItems`
  - `RolloutCmd::Persist`
  - `RolloutCmd::Flush`
  - `RolloutCmd::Shutdown`
- `codex-rs/thread-store/src/store.rs`
  - canonical append methods and metadata methods.

**How Codex uses it:**

Rollouts are append-only JSONL history. Metadata and list/read projections are
stored separately so the UI can list threads, show previews, paginate turns,
and read items without replaying every raw event each time.

**From-scratch implementation rule:**

Use two storage layers:

1. Canonical history:
   - append-only;
   - replayable;
   - durable after flush;
   - contains enough information to reconstruct conversation state.
2. Read model/projection:
   - indexed by thread ID;
   - optimized for list/read/search;
   - safe to rebuild from canonical history if needed.

### 14. Background Worker / Actor-Like Task

**Problem solved:** Request handlers should not directly own every long-running
loop. Transport routing, outbound fanout, rollout writing, and session
submission dispatch each need independent lifecycles.

**Where it appears:**

- `codex-rs/app-server/src/lib.rs`
  - outbound router task receives `OutgoingEnvelope` values;
  - processor task receives `TransportEvent` values.
- `codex-rs/core/src/session/handlers.rs`
  - `submission_loop(...)` receives submitted `Op` values.
- `codex-rs/rollout/src/recorder.rs`
  - writer task consumes `RolloutCmd` values.

**How Codex uses it:**

Several subsystems behave like actors:

- they own state;
- they receive messages through channels;
- they mutate state sequentially where needed;
- they report results through responses, events, callbacks, or acknowledgements.

**From-scratch implementation rule:**

Use background tasks for:

- accepting transport messages;
- routing outbound messages;
- driving each active thread/session;
- writing durable history;
- executing long-running commands/processes;
- watching filesystem or plugin state.

Each task needs a shutdown path and a way to surface terminal errors.

### 15. Router Registry

**Problem solved:** Exec-server exposes many filesystem and process methods.
A long `match` can work, but a registry makes method registration explicit and
keeps typed param/result conversion at the boundary.

**Where it appears:**

- `codex-rs/exec-server/src/rpc.rs`
  - `RpcRouter<S>`
  - `request(...)`
  - `request_with_id(...)`
  - `notification(...)`
  - route maps for request and notification methods.

**How Codex uses it:**

The router stores method names in `HashMap<&'static str, RequestRoute<S>>` and
`HashMap<&'static str, NotificationRoute<S>>`. Each registration deserializes
params into a typed payload, calls a handler, and serializes the result.

**From-scratch implementation rule:**

For utility RPC servers, implement:

```text
router.request("fs/readFile", read_file_handler)
router.request("process/start", process_start_handler)
router.notification("initialized", initialized_handler)
```

Keep deserialization at the router boundary, not inside business logic.

### 16. Transport Substitution

**Problem solved:** The same app-server behavior should be usable over stdio,
websocket, Unix socket, and in-process channels.

**Where it appears:**

- `codex-rs/app-server/src/in_process.rs`
  - module docs say it runs existing `MessageProcessor` and outbound routing
    on Tokio tasks, replacing socket/stdio transports with bounded in-memory
    channels.
  - typed incoming `ClientRequest` values still receive JSON-RPC result
    envelopes.

**How Codex uses it:**

The in-process runtime does not create a second execution contract. It reuses
the same processor and outgoing semantics as socket transports. That keeps TUI
or local embedders aligned with rich-client app-server behavior.

**From-scratch implementation rule:**

Design the server around a transport-neutral processor:

```text
Transport adapter -> incoming request enum -> shared processor -> outgoing envelope
```

Then add adapters:

- stdio adapter;
- websocket adapter;
- Unix socket adapter;
- in-process channel adapter.

## Low-Level Implementation Patterns

### Correlation IDs

**Where it appears:**

- `ConnectionRequestId` in `outgoing_message.rs` combines `connection_id` and
  request ID.
- `OutgoingMessageSender` tracks unresolved request contexts.
- Server-to-client requests use `next_server_request_id` and callback maps.

**What to implement:**

Every request, response, error, notification, and server-initiated callback
needs enough IDs to answer:

- Which connection sent this?
- Which request does this response complete?
- Which thread/turn/item does this event belong to?
- Which server-to-client request is waiting for an answer?

### Direct Response Versus Late Notification

**Where it appears:**

- `MessageProcessor::handle_initialized_client_request(...)` returns
  `Result<Option<ClientResponsePayload>, JSONRPCErrorError>`.
- `Ok(Some(response))` becomes a direct response.
- `Ok(None)` means the handler already arranged async completion or
  notification behavior.

**What to implement:**

Use three handler outcomes:

| Outcome | Meaning |
|---|---|
| `Response(payload)` | Complete request now. |
| `Accepted` | Request accepted; later events will follow. |
| `Error(error)` | Complete request with error now. |

### Shared Read Versus Exclusive Write

**Where it appears:**

- `RequestSerializationAccess::SharedRead`
- `RequestSerializationAccess::Exclusive`
- `RequestSerializationQueues::drain(...)` batches consecutive shared reads.

**What to implement:**

For each resource key, allow:

- many reads together;
- one write at a time;
- FIFO ordering between writes and reads.

This gives better concurrency than a single global mutex.

### Bounded Channels and Backpressure

**Where it appears:**

- `in_process.rs` documents bounded queues, `try_send`, `WouldBlock`, and
  overloaded errors.
- `app-server/src/transport.rs` defines the shared channel capacity used by
  in-process queues.

**What to implement:**

All high-volume channels should be bounded. Decide per channel what happens
when full:

- reject request with overload error;
- drop non-critical notification;
- block briefly with timeout;
- disconnect a broken peer.

Do not allow unbounded request or event queues in an agent runtime.

### Lifecycle Cleanup

**Where it appears:**

- `MessageProcessor::connection_closed(...)` cleans outgoing contexts,
  filesystem watches, command exec state, process exec state, and thread
  listeners.
- `RolloutRecorder` has `Flush` and `Shutdown` commands.
- `CodexThread` exposes `shutdown_and_wait` and `wait_until_terminated`.

**What to implement:**

Every resource owner needs cleanup hooks:

- connection closed;
- thread ended;
- process terminated;
- server shutdown;
- writer flush;
- pending approval canceled.

### Error Envelope

**Where it appears:**

- `OutgoingMessageSender::send_error(...)`.
- `MessageProcessor` converts dispatch errors into JSON-RPC errors.
- `exec-server/src/rpc.rs` has `RpcServerOutboundMessage::Error`.

**What to implement:**

Use one error envelope per protocol boundary:

```text
{
  id,
  error: {
    code,
    message,
    data
  }
}
```

Do not return arbitrary strings from some handlers and structured errors from
others.

## How Patterns Combine In One Turn

The `turn/start` flow uses most of the patterns together:

1. Client sends JSON-RPC-like method `turn/start`.
2. Transport receives `JSONRPCRequest`.
3. `MessageProcessor::process_request` records request context.
4. `deserialize_client_request` converts JSON to `ClientRequest::TurnStart`.
5. Dispatcher checks connection initialization.
6. Dispatcher checks experimental/capability requirements.
7. `ClientRequest::serialization_scope()` returns a thread-scoped queue key.
8. `RequestSerializationQueues` serializes this turn with other mutations for
   the same thread.
9. `handle_initialized_client_request` delegates to turn/thread processors.
10. Processor converts request params into a core `Op::UserInput`.
11. `CodexThread::submit(Op)` sends the command into the core session.
12. `submission_loop` dispatches the `Op`.
13. Core emits `EventMsg` values.
14. Event handlers persist canonical items through `ThreadStore` and
   `RolloutRecorder`.
15. App-server projects events into notifications.
16. `OutgoingMessageSender` sends direct responses and streamed
   notifications back to the correct connection(s).

This is the system's core shape:

```text
method string
  -> typed request
  -> scoped routing
  -> domain processor
  -> Op
  -> session loop
  -> EventMsg
  -> storage append/projection
  -> response/notification
```

## Minimal Pattern Set To Build First

If building from scratch, implement these patterns in this order:

1. **Typed protocol boundary**
   - Define request/response/notification structs.
   - Define one request enum.

2. **Thin transport adapter**
   - Read JSON messages.
   - Convert to typed request enum.
   - Write response/error/notification envelopes.

3. **Domain processor split**
   - Implement `ThreadProcessor` and `TurnProcessor` first.
   - Add config/filesystem/process processors later.

4. **Thread facade**
   - Implement `ThreadHandle.submit(Op)`.
   - Hide the internal session loop.

5. **Command/event split**
   - Define `Op`.
   - Define `EventMsg`.
   - Implement a session loop that receives `Op` and emits `EventMsg`.

6. **Append-only history**
   - Store events/items in JSONL.
   - Add flush semantics.

7. **Read projection**
   - Add thread metadata and list/read APIs.

8. **Resource-scoped serialization**
   - Add after the first shared mutable resources exist.

9. **Background worker cleanup**
   - Add shutdown, connection cleanup, pending request cancellation, and writer
     termination.

10. **Transport substitution**
    - Add in-process transport after the socket/stdio path is stable.

## Anti-Patterns Codex Avoids

| Anti-pattern | Why it is dangerous | Codex counter-pattern |
|---|---|---|
| Raw JSON inside domain handlers | Easy to miss fields and break clients silently. | Typed `ClientRequest`, params, responses. |
| One giant app-server handler | Every API change conflicts with every other API. | Domain processors. |
| One global request lock | Safe but unnecessarily slow. | Resource-scoped serialization. |
| Blocking `turn/start` until the model finishes | Bad UX and fragile network behavior. | Immediate response plus notifications. |
| UI directly mutates agent state | UI and core become tightly coupled. | UI sends requests; core receives `Op`; events update UI. |
| Storing only final assistant text | Cannot replay, debug, resume, or inspect tool calls. | Append-only rollout history. |
| Projection as source of truth | Metadata bugs can corrupt canonical history. | Canonical history plus rebuildable projections. |
| Separate semantics for in-process clients | Local clients drift from remote clients. | Shared `MessageProcessor`. |
| Unbounded queues | Memory grows under slow clients or runaway event streams. | Bounded channels/backpressure. |
| No cleanup on disconnect | Process handles, watches, callbacks, and listeners leak. | Connection cleanup hooks. |

## Beginner Design Checklist

When designing a similar system from scratch, answer these questions before
coding each layer:

1. What are the external methods?
2. What typed params and responses does each method use?
3. Which methods are long-running and need notifications?
4. Which resource does each method mutate?
5. Does that resource need exclusive serialization?
6. Which processor owns the method?
7. Does the request become a core command?
8. Which event types can the command emit?
9. Which events must be persisted to canonical history?
10. Which fields are projected for fast UI reads?
11. What cleanup is required if the connection closes?
12. What response/error envelope completes the original request?

## Practical Reimplementation Shape

A minimal implementation can be decomposed into these modules:

```text
cli/
  main.rs                         # thin command router

protocol/
  rpc.rs                          # request/response/error envelopes
  app.rs                          # ClientRequest, params, responses, notifications
  core.rs                         # Op, EventMsg

server/
  transport_stdio.rs              # JSON lines or stdio transport
  message_processor.rs            # initialize gate, typed dispatch
  outgoing.rs                     # response/error/notification sender
  serialization.rs                # resource-scoped queues
  processors/thread.rs            # thread/read/list/start
  processors/turn.rs              # turn/start/interrupt

core/
  thread_manager.rs               # composition root and active thread map
  thread.rs                       # ThreadHandle facade
  session_loop.rs                 # Op receiver and EventMsg sender

storage/
  store.rs                        # ThreadStore trait
  local.rs                        # JSONL + metadata implementation
  rollout_writer.rs               # append-only writer task

exec_server/
  rpc_router.rs                   # method registry
  fs.rs                           # filesystem handlers
  process.rs                      # process handlers
```

The first working version only needs:

- `thread/start`;
- `turn/start`;
- `thread/read`;
- append-only history;
- a simple metadata projection;
- direct responses;
- turn lifecycle notifications.

After that, add interrupts, approvals, filesystem/process APIs, plugins,
skills, MCP, remote execution, and daemon lifecycle management.

## Pattern Summary

Codex is not designed around one giant loop or one giant API handler. It is
designed around a small set of repeatable patterns:

- surfaces route into typed protocols;
- typed requests route into resource processors;
- long-running agent work becomes core commands;
- core emits events;
- events are persisted and projected;
- clients receive direct responses plus notifications;
- concurrency is controlled by resource-scoped queues;
- storage, execution, models, and tools sit behind adapter boundaries.

Those patterns are the reusable blueprint. The specific crates are Codex's
implementation of that blueprint.
