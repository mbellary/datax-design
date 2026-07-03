# Layer 5 - Core Data Models

This layer answers: what data models exist, who owns them, who mutates them,
where they are stored, and how they are surfaced back to clients or the UI.

## Source Map

Primary source files for this layer:

- `codex-rs/app-server-protocol/src/protocol/common.rs`
- `codex-rs/app-server-protocol/src/protocol/v2/thread.rs`
- `codex-rs/app-server-protocol/src/protocol/v2/thread_data.rs`
- `codex-rs/app-server-protocol/src/protocol/v2/turn.rs`
- `codex-rs/app-server-protocol/src/protocol/v2/item.rs`
- `codex-rs/protocol/src/protocol.rs`
- `codex-rs/protocol/src/user_input.rs`
- `codex-rs/protocol/src/models.rs`
- `codex-rs/thread-store/src/store.rs`
- `codex-rs/thread-store/src/types.rs`
- `codex-rs/thread-store/src/live_thread.rs`
- `codex-rs/thread-store/src/local/mod.rs`
- `codex-rs/rollout/src/recorder.rs`
- `codex-rs/exec-server-protocol/src/protocol.rs`
- `codex-rs/tools/src/tool_spec.rs`
- `codex-rs/tools/src/tool_call.rs`
- `codex-rs/tools/src/tool_output.rs`

## The Main Rule

Codex does not have one giant model object. It has model layers:

1. Client-facing RPC models in `codex-app-server-protocol`.
2. Core runtime command/event models in `codex-protocol`.
3. Model-provider input/output models in `codex-protocol` and `codex-api`.
4. Persistent history and metadata models in `codex-thread-store` and
   `codex-rollout`.
5. Live execution models in `codex-exec-server-protocol`.
6. Tool/plugin/MCP models around the core runtime.

This split is deliberate. A client can speak in terms of `thread/start`,
`turn/start`, and `ThreadItem`, while core can execute `Op::UserInput`, emit
`EventMsg`, and persist canonical `RolloutItem` records.

## Model Ownership Map

| Model group | Concrete types | Owner crate | Who creates it | Who mutates it | Where stored |
|---|---|---|---|---|---|
| RPC envelope | `ClientRequest`, `ServerNotification` | `codex-app-server-protocol` | JSON-RPC clients, in-process app clients | app-server transport/processor wraps and unwraps | Not durable by itself |
| Thread API object | `Thread`, `Turn`, `ThreadItem` | `codex-app-server-protocol` | app-server thread/turn handlers | app-server projections from core/store events | Returned to clients; history comes from thread store |
| Thread start payload | `ThreadStartParams`, `ThreadStartResponse` | `codex-app-server-protocol` | rich clients, TUI/exec in-process clients | app-server validates/defaults and starts session | Selected values become session metadata/history |
| Turn start payload | `TurnStartParams`, `TurnStartResponse` | `codex-app-server-protocol` | clients | app-server converts to core operation | User input and resulting events are persisted |
| Core input operation | `Op` | `codex-protocol` | app-server/TUI/exec bridges | core session consumes it | Not directly stored; effects are stored |
| Core output event | `Event`, `EventMsg` | `codex-protocol` | core session/tasks/tools | core emits events; app-server maps to notifications | Some events become `RolloutItem::EventMsg` |
| Model item | `ResponseItem` | `codex-protocol` | model API stream parsing and tool outputs | core appends/feeds back to model | `RolloutItem::ResponseItem` |
| User input item | `UserInput` | `codex-protocol` and API projection in app-server protocol | UI/client/user | converted across API/runtime boundary | Persisted as user message / rollout history |
| Persistent history item | `RolloutItem`, `RolloutLine` | `codex-protocol`, written by `codex-rollout` | core/live thread | append-only writes | JSONL rollout files |
| Thread storage API | `ThreadStore`, `LiveThread` | `codex-thread-store` | core session startup/resume | append metadata sync and explicit metadata APIs | local JSONL plus SQLite metadata index |
| Stored thread metadata | `CreateThreadParams`, `ResumeThreadParams`, `ThreadPersistenceMetadata`, `ThreadMetadataPatch` | `codex-thread-store` | core/app-server storage layer | `update_thread_metadata`, archive/delete/unarchive | SQLite and rollout compatibility records |
| Process/filesystem RPC | `ExecParams`, `ReadResponse`, `FsReadFileParams`, `ByteChunk` | `codex-exec-server-protocol` | core/app-server utility APIs | exec-server process/fs handlers | Live process table; not conversation history unless represented as items |
| Tool schema/call/output | `ToolSpec`, `ToolCall`, `ToolOutput` | `codex-tools` plus core tool router | tool registry and model calls | tool executors produce outputs | Tool calls/results become response items or thread items |

## App-Server RPC Envelope Models

The app-server protocol defines the JSON-RPC shaped boundary. In
`protocol/common.rs`, the `client_request_definitions!` macro expands into
`ClientRequest`, which is tagged by `method` and contains a request id plus a
typed `params` payload. The same file defines `ServerNotification` as a
`method` plus `params` enum for server-to-client streaming notifications.

Examples:

- `ThreadStart => "thread/start"` uses `v2::ThreadStartParams` and returns
  `v2::ThreadStartResponse`.
- `TurnStart => "turn/start"` uses `v2::TurnStartParams` and returns
  `v2::TurnStartResponse`.

Design from scratch:

- Define one transport envelope: `Request { id, method, params }`.
- Define one notification envelope: `Notification { method, params }`.
- Keep each method payload typed and versioned.
- Do not pass raw JSON maps deep into the system except at the serialization
  edge.

## Thread, Turn, and Item API Models

The client-facing conversation object is in
`app-server-protocol/src/protocol/v2/thread_data.rs`.

`Thread` includes:

- `id`
- `session_id`
- `forked_from_id`
- `parent_thread_id`
- `preview`
- `ephemeral`
- `history_mode`
- `model_provider`
- `created_at`
- `updated_at`
- `recency_at`
- `status`
- `path`
- `cwd`
- `cli_version`
- `source`
- `thread_source`
- `git_info`
- `name`
- `turns`

`Turn` includes:

- `id`
- `items`
- `items_view`
- `status`
- `error`
- `started_at`
- `completed_at`
- `duration_ms`

`ThreadItem` is the API projection of what clients render. Its variants include
user messages, hook prompts, agent messages, plans, reasoning, command
executions, file changes, MCP tool calls, dynamic tool calls, collaboration
agent tool calls, subagent activity, and web search items.

Important distinction:

- `ThreadItem` is for client display and app-server history APIs.
- `ResponseItem` is closer to the model-provider item stream.
- `RolloutItem` is the persisted replay record.

Design from scratch:

- Make `Thread` and `Turn` stable client projections.
- Make `Item` a tagged union.
- Do not make UI read raw model-provider event shapes directly.

## Thread Start Model

`ThreadStartParams` in `v2/thread.rs` is the creation payload. It carries
runtime configuration such as:

- `model`
- `model_provider`
- `service_tier`
- `cwd`
- `runtime_workspace_roots`
- `approval_policy`
- `approvals_reviewer`
- `sandbox`
- `permissions`
- `config`
- `base_instructions`
- `developer_instructions`
- `personality`
- `ephemeral`
- `history_mode`
- `session_start_source`
- `thread_source`
- `environments`
- `dynamic_tools`
- `selected_capability_roots`

`ThreadStartResponse` returns the created `Thread` plus the effective runtime
choices, including model, provider, service tier, cwd, workspace roots,
instruction sources, approval policy, reviewer, and sandbox policy.

Design from scratch:

- Treat start params as desired configuration.
- Return effective configuration after defaults, policy checks, and provider
  selection.
- Persist only the durable subset needed to resume the thread.

## Turn Start Model

`TurnStartParams` in `v2/turn.rs` starts actual agent work. It carries:

- `thread_id`
- `client_user_message_id`
- `input`
- `responsesapi_client_metadata`
- `additional_context`
- `environments`
- `cwd`
- `runtime_workspace_roots`
- `approval_policy`
- `approvals_reviewer`
- `sandbox_policy`
- `permissions`
- `model`
- `service_tier`
- `effort`
- `summary`
- `personality`
- `output_schema`
- `collaboration_mode`

`TurnStartResponse` returns a `Turn`.

Design from scratch:

- Separate thread creation from turn creation.
- Allow turn-scoped overrides.
- Convert a client `turn/start` request into an internal runtime command.

## User Input Model

The runtime `UserInput` enum is in `codex-rs/protocol/src/user_input.rs`.
Variants include:

- `Text { text, text_elements }`
- `Image { image_url, detail }`
- `LocalImage { path, detail }`
- `Skill { name, path }`
- `Mention { name, path }`

`MAX_USER_INPUT_TEXT_CHARS` caps a single text input at `1 << 20` characters.
That is a concrete protection against unbounded user-provided context.

Design from scratch:

- Treat user input as a tagged union, not just a string.
- Support text first, then images/files/mentions as additional input variants.
- Put hard size limits on every user-provided model-context fragment.

## Core Runtime Operation Model

`Op` in `codex-rs/protocol/src/protocol.rs` is the command enum that core
sessions consume. Important variants include:

- `Interrupt`
- `CleanBackgroundTerminals`
- realtime conversation variants
- `UserInput { items, final_output_json_schema, responsesapi_client_metadata,
  additional_context, thread_settings }`
- `ThreadSettings`
- `InterAgentCommunication`
- `ExecApproval`
- `PatchApproval`
- `ResolveElicitation`
- `UserInputAnswer`
- `RequestPermissionsResponse`

This is the runtime input model after app-server/TUI/exec have converted their
surface-specific request into core intent.

Design from scratch:

- Define an internal command enum for the agent engine.
- Keep it independent of JSON-RPC method names.
- Make approval, interruption, and tool-resolution first-class commands.

## Core Runtime Event Model

`Event` has an `id` and an `EventMsg`. `EventMsg` is the core output enum.
Important variants include:

- `Error`
- `Warning`
- `GuardianWarning`
- realtime events
- `ModelReroute`
- `ModelVerification`
- `TurnModerationMetadata`
- `SafetyBuffering`
- `ContextCompacted`
- `ThreadRolledBack`
- `TurnStarted`
- `ThreadSettingsApplied`
- `TurnComplete`
- `TokenCount`
- `AgentMessage`
- `UserMessage`
- reasoning events
- `SessionConfigured`
- `ThreadGoalUpdated`

These events are the bridge between the engine and all surfaces. TUI can render
them directly or indirectly. App-server can convert them into v2 notifications
and `ThreadItem` updates.

Design from scratch:

- Stream events from the engine as the single source of runtime truth.
- Correlate events with submission/turn ids.
- Convert events to UI notifications at the boundary.

## Model-Provider Item Model

`ResponseItem` in `codex-rs/protocol/src/models.rs` represents model-visible
and model-produced items. Variants include:

- `Message`
- `AgentMessage`
- `Reasoning`
- `LocalShellCall`
- `FunctionCall`
- `ToolSearchCall`
- `FunctionCallOutput`
- `CustomToolCall`

This is not the same as app-server `ThreadItem`. `ResponseItem` is closer to
the Responses API conversation stream and is used for model interaction and
durable replay.

Design from scratch:

- Keep provider-shaped message/call/output items separate from UI-shaped
  display items.
- Store enough provider-shaped history to resume without reconstructing from
  UI-only text.

## Persistent History Model

`RolloutItem` and `RolloutLine` are in `codex-rs/protocol/src/protocol.rs`.
`RolloutItem` variants include:

- `SessionMeta`
- `ResponseItem`
- `InterAgentCommunication`
- `InterAgentCommunicationMetadata`
- `Compacted`
- `TurnContext`
- `WorldState`
- `EventMsg`

`RolloutLine` stores a timestamp plus a flattened `RolloutItem`. This is the
JSONL replay format written by `codex-rollout`.

`RolloutRecorder` in `codex-rs/rollout/src/recorder.rs` owns a background
writer. Its command enum includes:

- `AddItems`
- `Persist`
- `Flush`
- `Shutdown`

Design from scratch:

- Use append-only JSONL or an equivalent event log for replayable history.
- Put timestamps on durable lines.
- Persist runtime context snapshots such as cwd/model/sandbox, not only
  messages.

## Thread Store Model

`ThreadStore` in `codex-rs/thread-store/src/store.rs` is the persistence trait.
Important methods:

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
- `search_threads`
- `list_turns`
- `list_items`
- `update_thread_metadata`
- `archive_thread`
- `unarchive_thread`
- `delete_thread`

`LiveThread` wraps an active thread writer. Its `append_items` method writes
items through `ThreadStore::append_items`, then observes canonical items and
may call `update_thread_metadata`.

This creates two update paths:

1. History updates: append rollout items.
2. Metadata updates: apply patches to thread metadata.

Design from scratch:

- Define a storage trait before writing local storage code.
- Keep append-only history separate from mutable metadata.
- Make live sessions write through one active `LiveThread` handle.

## Local Storage Model

`LocalThreadStore` is documented as a filesystem/SQLite implementation.
Its comments state the storage contract:

- Rollout JSONL files are the durable replay format.
- SQLite is the queryable metadata index for list/read paths.
- Live appends still write canonical JSONL history.
- Append-derived metadata is observed above the store and applied through
  `ThreadStore::update_thread_metadata`.

`CreateThreadParams` contains the durable startup metadata: session id, thread
id, fork/parent ids, source, thread source, originator, base instructions,
dynamic tools, selected capability roots, multi-agent version, history mode,
initial window id, and `ThreadPersistenceMetadata`.

`ThreadPersistenceMetadata` contains:

- `cwd`
- `model_provider`
- `memory_mode`

Design from scratch:

- Store replay history and query indexes separately.
- Make the replay file useful even if the index is missing or corrupt.
- Keep create/resume params explicit so storage can be implemented locally or
  remotely.

## Execution and Filesystem Models

`codex-exec-server-protocol` owns live subprocess/filesystem RPC payloads.

`ExecParams` includes:

- `process_id`
- `argv`
- `cwd`
- `env_policy`
- `env`
- `tty`
- `pipe_stdin`
- `arg0`
- `sandbox`
- `enforce_managed_network`
- `managed_network`

`ReadResponse` includes chunks, next sequence, exit state, exit code, closed
state, failure, and sandbox-denial classification.

`ByteChunk` stores bytes with base64 serialization. This keeps binary process
output and filesystem reads JSON-compatible.

Design from scratch:

- Keep live process I/O outside the conversation model.
- Use sequence numbers for streaming process output.
- Convert execution results into conversation/thread items only when they need
  to be shown or persisted as agent work.

## Tool Models

The shared tool crate defines model-visible tool descriptors and runtime tool
calls.

`ToolSpec` variants include:

- function tools
- namespace tools
- tool search
- image generation
- web search
- freeform custom tools

`ToolCall` carries the runtime invocation context:

- `turn_id`
- `call_id`
- `tool_name`
- `model`
- `truncation_policy`
- `conversation_history`
- `turn_item_emitter`
- `environments`
- `payload`

`ToolOutput` converts a tool result back to a model response item through
`to_response_item`.

Design from scratch:

- Separate tool schema from tool invocation.
- Give tool execution access to turn id, call id, history, environment, and an
  emitter for UI-visible progress.
- Make tool outputs responsible for converting themselves back into
  model-consumable output.

## Canonical Request-to-Data Flow

The common data path for a normal user turn is:

1. Client sends `turn/start` with `TurnStartParams`.
2. App-server receives a typed `ClientRequest::TurnStart`.
3. App-server converts it to core intent, primarily `Op::UserInput`.
4. Core session starts a turn task.
5. Core streams `Event { id, msg: EventMsg }`.
6. Model/provider events are represented as `ResponseItem` values.
7. Persistent replay records are appended as `RolloutItem` values.
8. `LiveThread::append_items` writes through `ThreadStore::append_items`.
9. Append-derived metadata may trigger `ThreadStore::update_thread_metadata`.
10. App-server surfaces updates as `ServerNotification` and as updated
    `Thread`, `Turn`, or `ThreadItem` projections.

## What You Need to Build This From Scratch

Minimum model set:

- `ThreadId`, `TurnId`, `ItemId`, `SessionId`
- `Thread`
- `Turn`
- `Item` tagged union
- `UserInput` tagged union
- `StartThreadParams`
- `StartTurnParams`
- `RuntimeOp`
- `RuntimeEvent`
- `ModelItem`
- `ToolSpec`
- `ToolCall`
- `ToolOutput`
- `RolloutItem`
- `RolloutLine`
- `ThreadMetadata`
- `ThreadMetadataPatch`
- `ProcessExecParams`
- `ProcessOutputChunk`
- `RequestEnvelope`
- `ResponseEnvelope`
- `NotificationEnvelope`

Implementation order:

1. Define ids and transport envelopes.
2. Define `Thread`, `Turn`, and `Item` as API projections.
3. Define internal `RuntimeOp` and `RuntimeEvent`.
4. Define model/provider items separately from UI items.
5. Define append-only `RolloutItem` and `RolloutLine`.
6. Define a `ThreadStore` interface.
7. Implement local JSONL history writing.
8. Add mutable metadata indexing.
9. Add process/filesystem RPC models.
10. Add tool schema/call/output models.

The important design constraint is not the exact Rust types. It is the boundary:
API models, runtime models, provider models, and persistence models must be
related but not collapsed into one mutable object.
