# API and Request Routing

This artifact explains how requests and responses enter the system, how they
are routed, and which payload styles are involved.

## Major API Boundaries

| Boundary | Transport | Payload shape | Primary crates |
| --- | --- | --- | --- |
| CLI | process argv/stdin/stdout | Clap command structs, terminal I/O | `cli`, `tui`, `exec` |
| app-server | JSON-RPC over stdio, websocket, Unix socket, or off | typed app-server protocol DTOs | `app-server`, `app-server-protocol`, `app-server-transport` |
| exec-server | JSON-RPC over websocket or remote relay | typed exec-server protocol DTOs | `exec-server`, `exec-server-protocol` |
| core agent | in-process Rust calls/events | `Op`, `EventMsg`, task inputs, rollout items | `core`, `protocol` |
| thread storage | in-process trait API | thread/history/metadata params and results | `thread-store`, `rollout`, `state` |
| model/backend | HTTP/SSE or provider-specific clients | Responses API/model-provider payloads | `backend-client`, `codex-api`, `model-provider` |
| MCP/extensions | MCP protocol and internal extension APIs | tool/resource DTOs | `codex-mcp`, `mcp-server`, `ext/*` |

## App-Server Request Path

High-level path:

```text
client
  -> app-server transport
  -> TransportEvent::IncomingMessage
  -> JSONRPCMessage::Request
  -> MessageProcessor::process_request
  -> deserialize_client_request
  -> MessageProcessor::handle_client_request
  -> specific processor
  -> response or notification via OutgoingMessageSender
  -> outbound router
  -> client connection writer
```

The app-server processor loop receives transport events. For incoming JSON-RPC
requests it calls `processor.process_request`. That method:

1. builds a connection-aware request id
2. creates a tracing span
3. deserializes the raw JSON-RPC request into a typed `ClientRequest`
4. calls `handle_client_request`
5. sends a JSON-RPC error if routing or handling fails

Evidence:

- `codex-rs/app-server/src/lib.rs:1014` handles incoming messages.
- `codex-rs/app-server/src/lib.rs:1016` matches
  `JSONRPCMessage::Request`.
- `codex-rs/app-server/src/lib.rs:1023` calls `processor.process_request`.
- `codex-rs/app-server/src/message_processor.rs:591` defines
  `process_request`.
- `codex-rs/app-server/src/message_processor.rs:619` deserializes the request.
- `codex-rs/app-server/src/message_processor.rs:626` calls
  `handle_client_request`.
- `codex-rs/app-server/src/message_processor.rs:637` sends errors.

## App-Server Dispatch Table

App-server dispatch is a typed `match` over `ClientRequest`. Examples:

| Incoming method | Typed variant | Handler |
| --- | --- | --- |
| `thread/start` | `ClientRequest::ThreadStart` | `thread_processor.thread_start` |
| `thread/resume` | `ClientRequest::ThreadResume` | `thread_processor.thread_resume` |
| `thread/fork` | `ClientRequest::ThreadFork` | `thread_processor.thread_fork` |
| `thread/list` | `ClientRequest::ThreadList` | `thread_processor.thread_list` |
| `thread/read` | `ClientRequest::ThreadRead` | `thread_processor.thread_read` |
| `thread/items/list` | `ClientRequest::ThreadItemsList` | `thread_processor.thread_items_list` |
| `turn/start` | `ClientRequest::TurnStart` | `turn_processor.turn_start` |
| `thread/settings/update` | `ClientRequest::ThreadSettingsUpdate` | `turn_processor.thread_settings_update` |
| `skills/list` | `ClientRequest::SkillsList` | `catalog_processor.skills_list` |
| `plugin/list` | `ClientRequest::PluginList` | `plugin_processor.plugin_list` |
| `model/list` | `ClientRequest::ModelList` | `catalog_processor.model_list` |
| `command/exec` | `ClientRequest::OneOffCommandExec` | command exec processor |
| process APIs | `ClientRequest::Process*` | process exec processor |

Evidence:

- `codex-rs/app-server/src/message_processor.rs:1093` routes
  `ThreadStart`.
- `codex-rs/app-server/src/message_processor.rs:1110` routes
  `ThreadResume`.
- `codex-rs/app-server/src/message_processor.rs:1122` routes `ThreadFork`.
- `codex-rs/app-server/src/message_processor.rs:1214` routes `ThreadList`.
- `codex-rs/app-server/src/message_processor.rs:1223` routes `ThreadRead`.
- `codex-rs/app-server/src/message_processor.rs:1229` routes
  `ThreadItemsList`.
- `codex-rs/app-server/src/message_processor.rs:1323` routes `TurnStart`.
- `codex-rs/app-server/src/message_processor.rs:1175` routes
  `ThreadSettingsUpdate`.
- `codex-rs/app-server/src/message_processor.rs:1245` routes `SkillsList`.
- `codex-rs/app-server/src/message_processor.rs:1263` routes `PluginList`.
- `codex-rs/app-server/src/message_processor.rs:1304` routes `ModelList`.

## App-Server Response and Notification Path

There are two outbound concepts:

- responses: tied to a JSON-RPC request id
- notifications: pushed to initialized/subscribed clients

The outbound router decouples response generation from connection writes. It
receives envelopes from the processor side and calls `route_outgoing_envelope`
against the currently open outbound connections.

Evidence:

- `codex-rs/app-server/src/lib.rs:850` through
  `codex-rs/app-server/src/lib.rs:874` shows the outbound router loop.
- `codex-rs/app-server/src/lib.rs:866` receives outgoing envelopes.
- `codex-rs/app-server/src/lib.rs:870` calls `route_outgoing_envelope`.
- `codex-rs/app-server/src/lib.rs:1054` through
  `codex-rs/app-server/src/lib.rs:1078` marks a connection initialized and
  sends initialization-related notifications.
- `codex-rs/app-server/src/lib.rs:1121` through
  `codex-rs/app-server/src/lib.rs:1135` attaches initialized connections to
  newly created threads.

## Exec-Server Request Path

Exec-server has a separate protocol. It does not use app-server's
`ClientRequest` enum. Instead, it stores routes in `RpcRouter` maps keyed by
method string.

High-level path:

```text
client/harness
  -> exec-server JSON-RPC transport
  -> method string
  -> RpcRouter::request_route or notification_route
  -> decode params into typed Rust struct
  -> handler
  -> JSON-RPC response, error, or notification
```

The exec-server protocol constants define request and notification names. The
router generics enforce typed params and typed serialized results at each
registered route.

Evidence:

- `codex-rs/exec-server-protocol/src/protocol.rs:16` through
  `codex-rs/exec-server-protocol/src/protocol.rs:42` defines exec-server
  methods.
- `codex-rs/exec-server/src/rpc.rs:117` defines `request_routes`.
- `codex-rs/exec-server/src/rpc.rs:118` defines `notification_routes`.
- `codex-rs/exec-server/src/rpc.rs:138` registers request routes.
- `codex-rs/exec-server/src/rpc.rs:150` decodes request params.
- `codex-rs/exec-server/src/rpc.rs:160` serializes successful request results.
- `codex-rs/exec-server/src/rpc.rs:202` registers notification routes.
- `codex-rs/exec-server/src/rpc.rs:211` decodes notification params.
- `codex-rs/exec-server/src/rpc.rs:224` looks up request routes.

## Storage Request Path

Thread storage is behind the `ThreadStore` trait. The storage API is not HTTP or
JSON-RPC. It is an in-process Rust trait with typed params/results.

Important storage methods:

- `create_thread`
- `resume_thread`
- `append_items`
- `persist_thread`
- `flush_thread`
- `shutdown_thread`
- `discard_thread`
- `load_history`
- `read_thread`
- `list_threads`
- `list_turns`
- `list_items`
- `update_thread_metadata`
- `archive_thread`
- `unarchive_thread`
- `delete_thread`

The local implementation uses two durable surfaces:

- rollout JSONL files: canonical replay history
- SQLite state DB: queryable metadata index for list/read paths

`LiveThread` is the preferred active-session persistence wrapper. It delegates
history writes to `ThreadStore::append_items`, observes canonical appended items,
and applies metadata updates through `ThreadStore::update_thread_metadata`.

Evidence:

- `codex-rs/thread-store/src/store.rs:33` defines `ThreadStore`.
- `codex-rs/thread-store/src/store.rs:45` through
  `codex-rs/thread-store/src/store.rs:139` lists storage operations.
- `codex-rs/thread-store/src/local/mod.rs:45` documents local storage.
- `codex-rs/thread-store/src/local/mod.rs:47` identifies rollout JSONL as
  durable replay.
- `codex-rs/thread-store/src/local/mod.rs:50` identifies SQLite as queryable
  metadata.
- `codex-rs/thread-store/src/live_thread.rs:89` implements `LiveThread`.
- `codex-rs/thread-store/src/live_thread.rs:146` defines
  `LiveThread::append_items`.
- `codex-rs/thread-store/src/live_thread.rs:157` calls
  `ThreadStore::append_items`.
- `codex-rs/thread-store/src/live_thread.rs:169` observes appended items.
- `codex-rs/thread-store/src/live_thread.rs:175` calls
  `ThreadStore::update_thread_metadata`.

## Payload Types

| Boundary | Request payload | Response payload | Streaming/update payload |
| --- | --- | --- | --- |
| app-server | JSON-RPC request with typed `*Params` | JSON-RPC response with typed `*Response` | JSON-RPC notifications such as `thread/started`, `turn/started`, `item/*`, `turn/completed` |
| exec-server | JSON-RPC request with typed structs | JSON-RPC response with typed structs | notifications such as `process/output`, `process/exited`, `process/closed` |
| core session | Rust `TurnInput`, task structs, context structs | task result / `EventMsg` | `EventMsg` stream and lifecycle events |
| thread-store | Rust param structs | Rust result structs | durable rollout items and metadata patches |

## Design Lesson

For a from-scratch implementation, do not route everything through stringly
typed maps. Use string method names only at wire boundaries, immediately decode
them into typed request enums or typed route payloads, and keep business logic
behind explicit processors and storage traits.
