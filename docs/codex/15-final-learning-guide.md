# Final Learning Guide

This guide ties the architecture study together into one path. It does not
replace the detailed layer artifacts. It tells you what to read, what concepts
to keep in your head, which files prove each claim, and what to build first if
you were implementing a Codex-like system from scratch.

## Source-Backed Ground Rules

Every major claim in this guide points back to one of the completed artifacts
and to concrete source anchors recorded there. The most important anchors are:

| Area | Source anchors |
|---|---|
| CLI entry points | `codex-rs/cli/src/main.rs` |
| Rich-client app-server | `codex-rs/app-server/src/lib.rs`, `codex-rs/app-server/src/message_processor.rs`, `codex-rs/app-server/README.md` |
| App-server protocol | `codex-rs/app-server-protocol/src/protocol/common.rs`, `codex-rs/app-server-protocol/src/protocol/v2/thread.rs`, `codex-rs/app-server-protocol/src/protocol/v2/turn.rs`, `codex-rs/app-server-protocol/src/protocol/v2/thread_data.rs` |
| Request serialization | `codex-rs/app-server/src/request_serialization.rs` |
| Core runtime protocol | `codex-rs/protocol/src/protocol.rs` |
| Thread/session facade | `codex-rs/core/src/thread_manager.rs`, `codex-rs/core/src/codex_thread.rs`, `codex-rs/core/src/session/handlers.rs` |
| Core async tasks | `codex-rs/core/src/tasks/mod.rs` |
| Storage boundary | `codex-rs/thread-store/src/store.rs`, `codex-rs/thread-store/src/live_thread.rs`, `codex-rs/thread-store/src/local/mod.rs` |
| Rollout history | `codex-rs/rollout/src/recorder.rs` |
| State databases | `codex-rs/state/src/runtime.rs` |
| Exec-server | `codex-rs/exec-server/src/rpc.rs`, `codex-rs/exec-server/src/server/registry.rs`, `codex-rs/exec-server-protocol/src/protocol.rs` |

Use the detailed artifacts when you need exact method names, enum variants,
payload structs, and request paths.

## One-Page Mental Model

Codex is a local coding-agent runtime. It exposes multiple product surfaces:
terminal UI, scriptable CLI commands, a rich-client app-server, an app-server
daemon, an exec-server for process/filesystem work, MCP integration, plugins,
skills, SDKs, and cloud task support.

The central object is a thread. A thread contains turns. A turn contains items.
Items are user messages, assistant messages, tool calls, tool outputs, errors,
and lifecycle records. The system stores canonical thread history separately
from metadata and list/search projections.

Requests enter through a surface. Rich clients use app-server method strings
such as `thread/start` and `turn/start`. Terminal flows call core directly.
Exec-server clients use a separate JSON-RPC method catalog for process and
filesystem operations.

After transport parsing, app-server requests become typed `ClientRequest`
variants. Domain processors validate the request and either answer directly or
submit an `Op` into a core thread runtime. The core runtime emits `EventMsg`
values. Those events are persisted, converted into UI-facing items, and sent
back to clients as notifications such as `turn/started`, `item/started`,
`item/completed`, and `turn/completed`.

The most important split is this:

| Plane | What moves through it | Concrete examples |
|---|---|---|
| Request plane | Client commands into the system | `thread/start`, `turn/start`, `fs/readFile`, `process/spawn` |
| Event plane | Runtime progress out of the system | `EventMsg::TurnStarted`, `EventMsg::AgentMessage`, `EventMsg::TurnComplete` |
| Storage plane | Durable history and projections | rollout JSONL, SQLite metadata, `ThreadStore`, `LiveThread` |
| Execution plane | Tool and command work | exec-server process APIs, filesystem APIs, sandbox/approval handling |
| Extension plane | External capabilities | MCP, plugins, skills, apps/connectors |

If you understand those five planes, the rest of the codebase becomes a set of
specific implementations around them.

## Reading Path

Read in this order if you are new to this type of system:

1. `01-repository-orientation.md`
   - Goal: know where the major code lives.
   - Key source: `codex-rs/cli/src/main.rs`.

2. `04-product-surface-inventory.md`
   - Goal: know the user-facing and integration-facing products.
   - Key idea: this is not one API; it is several surfaces over shared agent
     runtime.

3. `05-crate-responsibility-map.md`
   - Goal: understand why the repo is split into many crates.
   - Key idea: CLI, protocol, core runtime, storage, server, execution, and
     extension crates have different responsibilities.

4. `06-primary-use-cases.md`
   - Goal: understand what the system is for.
   - Key idea: use cases explain why the architecture has threads, turns,
     events, storage, daemon mode, exec-server, MCP, plugins, and SDKs.

5. `07-core-data-models.md`
   - Goal: understand the nouns.
   - Key types: `Thread`, `Turn`, `ThreadItem`, `ThreadStartParams`,
     `TurnStartParams`, `UserInput`, `Op`, `Event`, `EventMsg`, `ResponseItem`,
     `RolloutItem`, `ThreadStore`, `LiveThread`.

6. `08-api-and-protocol-catalog.md`
   - Goal: understand the external and internal APIs.
   - Key boundaries: app-server JSON-RPC-like envelopes, typed app-server
     protocol, core `Op`/`EventMsg`, exec-server protocol, SDK JSONL, storage
     trait.

7. `09-end-to-end-request-flows.md`
   - Goal: trace real flows from request to response, notification, storage,
     and UI projection.
   - Key flows: `thread/start`, `turn/start`, TUI conversation, exec,
     filesystem requests, command/process requests, exec-server remote
     execution, MCP, plugins/skills/apps, daemon, SDK streaming.

8. `10-routing-and-handler-mechanics.md`
   - Goal: understand how requests find the right handler.
   - Key mechanisms: CLI subcommand dispatch, app-server transport events,
     request deserialization, `ClientRequest` dispatch, initialization gates,
     serialization queues, domain processors, outgoing envelopes, exec-server
     router registry.

9. `11-storage-and-update-paths.md`
   - Goal: understand what is stored, where, when, and how it surfaces.
   - Key mechanisms: `ThreadStore`, `LiveThread`, rollout JSONL, SQLite
     metadata/index, read/list projections, explicit metadata updates, derived
     metadata updates, archive/delete, config/auth/daemon side stores.

10. `02-runtime-processes-and-task-topology.md`
    - Goal: distinguish OS processes from in-process async tasks.
    - Key processes: `codex app-server`, `codex app-server daemon`,
      `codex exec-server`.
    - Key in-process tasks: app-server processor/outbound loops, session tasks.

11. `12-system-design-synthesis.md`
    - Goal: combine the pieces into one architecture.
    - Key idea: the system is a typed, event-driven, thread-centered local
      agent runtime.

12. `13-design-patterns.md`
    - Goal: learn the reusable patterns.
    - Key patterns: thin entrypoint router, typed protocol boundary, JSON
      envelope to typed request, resource-scoped serialization, domain
      processors, response plus notification stream, command/event split,
      port/adapter storage, append-only history plus projection, background
      workers, router registry.

13. `14-build-from-scratch-blueprint.md`
    - Goal: build your implementation plan.
    - Key output: exact phases from skeleton to protocol, storage, runtime,
      app-server, tools, exec-server, daemon, and extensions.

14. This file.
    - Goal: keep the map in one place while you study and implement.

`03-api-and-request-routing.md` is an older consolidated supporting artifact.
Use it after the dedicated API, flow, routing, and storage artifacts when you
want a compact cross-check.

## Mapping To The Original Questions

| Original question | Primary artifact | What to look for |
|---|---|---|
| What are the use cases? | `06-primary-use-cases.md` | Fifteen concrete workflows, entry points, handlers, payloads, responses, and storage effects |
| What data models are used? | `07-core-data-models.md` | API models, core runtime models, persistent models, execution models, tool models |
| What APIs are used? | `08-api-and-protocol-catalog.md` | App-server, core protocol, exec-server, SDK, storage, server-to-client requests |
| What is the system design? | `12-system-design-synthesis.md` | Component map, control/data planes, runtime modes, invariants |
| What processes are used? | `02-runtime-processes-and-task-topology.md` | OS processes, daemon-managed processes, server loops, in-process tasks |
| How are things stored and updated? | `11-storage-and-update-paths.md` | `ThreadStore`, rollout JSONL, SQLite metadata, side stores, live notifications |
| How do request/response flows work? | `09-end-to-end-request-flows.md` | Rich-client, CLI, TUI, exec-server, SDK, MCP, plugin, daemon flows |
| What high-level and low-level patterns are used? | `13-design-patterns.md` | Architectural patterns and implementation patterns |
| Which request/response methods and payload types are involved? | `08-api-and-protocol-catalog.md` | Method strings, `*Params`, `*Response`, notifications, core and exec-server payloads |
| How are requests and responses routed and handled? | `10-routing-and-handler-mechanics.md` | Dispatch tables, processors, serialization scopes, outgoing envelopes |
| Where is data updated, stored, and surfaced in the UI? | `11-storage-and-update-paths.md`, `09-end-to-end-request-flows.md` | Append paths, metadata paths, read/list paths, event-to-item projection |
| How do I design and implement it from scratch? | `14-build-from-scratch-blueprint.md` | Exact phases, minimal vertical slice, tests, failure modes, definition of done |

## The Core Runtime Flow

The central turn flow is:

```text
client request
  -> transport envelope
  -> typed ClientRequest
  -> request serialization scope
  -> domain processor
  -> core Op
  -> thread/session runtime
  -> model/tool work
  -> EventMsg stream
  -> append canonical history
  -> update metadata/projections
  -> direct response and notifications
  -> UI surfaces updated state
```

For a rich-client `turn/start`, the important concrete path is:

| Step | Code area | Artifact |
|---|---|---|
| Receive JSON-RPC-like request | `app-server/src/lib.rs` | `10-routing-and-handler-mechanics.md` |
| Deserialize method and params | `app-server/src/message_processor.rs` | `08-api-and-protocol-catalog.md` |
| Match typed request | `app-server-protocol/src/protocol/common.rs` | `08-api-and-protocol-catalog.md` |
| Serialize by thread scope | `app-server/src/request_serialization.rs` | `10-routing-and-handler-mechanics.md` |
| Process turn start | `app-server/src/turn_processor.rs` | `09-end-to-end-request-flows.md` |
| Submit `Op::UserInput` or related operation | `core/src/codex_thread.rs`, `protocol/src/protocol.rs` | `07-core-data-models.md` |
| Handle operation in session | `core/src/session/handlers.rs` | `10-routing-and-handler-mechanics.md` |
| Emit events | `protocol/src/protocol.rs` | `07-core-data-models.md` |
| Append thread items | `thread-store/src/live_thread.rs`, `thread-store/src/local/live_writer.rs` | `11-storage-and-update-paths.md` |
| Send notifications | `app-server/src/outgoing_message.rs` | `09-end-to-end-request-flows.md` |

The direct response to `turn/start` is not the full assistant answer. It is the
acknowledgement that a turn was accepted and started. The actual assistant
work arrives later as event notifications.

## The Data Model You Must Build First

A beginner implementation should start with these models and no more:

| Model | Purpose |
|---|---|
| `ThreadId` | Stable identifier for a conversation/work unit |
| `TurnId` | Stable identifier for one user request inside a thread |
| `ItemId` | Stable identifier for one stored/rendered event item |
| `Thread` | API projection for listing/reading a thread |
| `Turn` | API projection for turn status and lifecycle |
| `ThreadItem` | API projection for messages, tool calls, outputs, and errors |
| `ClientRequest` | Typed internal request enum after parsing wire methods |
| `ServerNotification` | Typed outgoing notification enum |
| `Op` | Command submitted into the core runtime |
| `EventMsg` | Event emitted by the runtime |
| `RolloutItem` | Durable append-only history item |
| `ThreadStore` | Storage trait/port |
| `LiveThread` | Write-through handle that appends history and updates metadata |

Do not start with plugin systems, daemon mode, or remote execution. Those are
later layers. The first milestone is one thread, one turn, one fake model
response, persisted history, and streamed notifications.

## The Minimal API You Must Build First

Start with this app-server subset:

| Method | Request payload | Response/notification |
|---|---|---|
| `initialize` | client info/capabilities | initialization response |
| `thread/start` | `ThreadStartParams` | `ThreadStartResponse`, `thread/started` |
| `turn/start` | `TurnStartParams` | `TurnStartResponse`, `turn/started`, `item/*`, `turn/completed` |
| `thread/read` | thread id | `ThreadReadResponse` |
| `thread/list` | cursor/limit | `ThreadListResponse` |

Then add:

| Method family | Why it comes later |
|---|---|
| `thread/update` and lifecycle methods | Requires metadata semantics |
| filesystem utility methods | Requires path policy and execution boundaries |
| command/process methods | Requires process lifecycle and cleanup |
| approval requests | Requires server-to-client request callbacks |
| exec-server methods | Requires transport split and remote/local abstraction |
| plugin/skill/MCP methods | Requires extension registry and trust model |

The app-server protocol uses `*Params` for request payloads, `*Response` for
responses, and `*Notification` for notifications. Keep that convention from
the beginning because it prevents ambiguous API shapes.

## The Minimal Storage You Must Build First

Use two storage forms:

1. Append-only history file per thread.
   - Stores canonical items in order.
   - Used for resume and audit.
   - Corresponds to Codex rollout JSONL.

2. Metadata/index database.
   - Stores thread list/search fields, archive/delete status, timestamps, and
     derived summaries.
   - Used for fast `thread/list`, `thread/read`, turn listing, and item paging.
   - Corresponds to Codex local SQLite metadata/index storage.

The write rule is:

```text
append canonical item
  -> flush or make durable enough for the storage contract
  -> update derived metadata/projection
  -> notify live subscribers
```

Do not make metadata the source of truth for history. Metadata is a projection.
The append-only history is the canonical record.

## The Minimal Runtime You Must Build First

Implement the runtime as an event-driven loop:

```text
Op::UserInput { thread_id, turn_id, input }
  -> emit EventMsg::TurnStarted
  -> emit EventMsg::UserMessage
  -> call model adapter
  -> emit EventMsg::AgentMessageDelta or AgentMessage
  -> emit EventMsg::TurnComplete
```

At first, the model adapter can be fake:

```text
input: "hello"
output: "fake assistant response"
```

The point is to prove the architecture before adding real model APIs, tools,
approvals, sandboxing, or remote execution.

## Request Routing Rules

Implement routing in layers:

1. Transport reads raw bytes or in-process messages.
2. Envelope parser extracts request id, method, and params.
3. Typed protocol maps method strings to `ClientRequest` variants.
4. Initialization gate rejects non-initialize calls before `initialize`.
5. Capability/experimental gate rejects unsupported methods or fields.
6. Serialization queue decides whether the request can run now.
7. Domain processor handles the typed request.
8. Outgoing router sends responses, errors, notifications, and server-to-client
   requests to the correct connection.

That layering is concrete in Codex through:

- `app-server/src/lib.rs` for transport and app-server loops.
- `app-server/src/message_processor.rs` for request processing.
- `app-server-protocol/src/protocol/common.rs` for typed request mapping.
- `app-server/src/request_serialization.rs` for scope-aware serialization.
- `app-server/src/outgoing_message.rs` for response/notification routing.

## Process And Task Rules

Do not confuse these:

| Runtime unit | Meaning |
|---|---|
| OS process | Separate executable launched by the operating system |
| Daemon-managed process | Long-running app-server controlled by daemon commands and pidfiles |
| Server loop | In-process async loop inside a process |
| Session task | In-process async unit that performs runtime work for a session/turn |
| Storage worker | In-process async or blocking worker that persists data |

The concrete Codex process map is:

| Process/task | Where to study |
|---|---|
| `codex` CLI/TUI process | `01-repository-orientation.md`, `04-product-surface-inventory.md` |
| `codex exec` process | `06-primary-use-cases.md`, `09-end-to-end-request-flows.md` |
| `codex app-server` process | `02-runtime-processes-and-task-topology.md`, `10-routing-and-handler-mechanics.md` |
| `codex app-server daemon` | `02-runtime-processes-and-task-topology.md` |
| `codex exec-server` | `02-runtime-processes-and-task-topology.md`, `08-api-and-protocol-catalog.md` |
| app-server processor/outbound loops | `10-routing-and-handler-mechanics.md` |
| core `SessionTask` work | `02-runtime-processes-and-task-topology.md`, `10-routing-and-handler-mechanics.md` |

If you are implementing from scratch, add a daemon only after your app-server
works as a normal foreground process.

## Design Patterns To Reuse

Use these patterns in this order:

| Pattern | Why it matters |
|---|---|
| Thin entrypoint router | Keeps CLI/app-server binaries from owning business logic |
| Typed protocol boundary | Prevents stringly typed request handling from spreading |
| JSON envelope to typed request | Lets transport stay generic while handlers stay typed |
| Initialization gate | Prevents clients from using the server before capabilities are known |
| Resource-scoped serialization | Prevents conflicting writes to the same thread/process/resource |
| Domain processor ownership | Keeps thread, turn, filesystem, process, MCP, and plugin logic separated |
| Response plus notification stream | Models long-running turns correctly |
| Thread/session facade | Gives callers one safe handle for submitting operations |
| Command/event split | Separates what clients ask for from what the runtime reports |
| Port/adapter storage | Lets runtime depend on `ThreadStore`, not a concrete database |
| Append-only history plus projection | Preserves auditability and fast reads |
| Router registry | Makes exec-server-style method registration explicit |
| Transport substitution | Supports socket, stdio, and in-process testing |

The detailed pattern evidence is in `13-design-patterns.md`.

## Build From Scratch In Exact Order

Use this implementation order:

1. Create a workspace with separate modules for protocol, app-server, core,
   storage, model adapter, exec adapter, and CLI.
2. Define a JSON-RPC-like envelope with request id, method, params, response,
   error, and notification forms.
3. Define typed request/response/notification enums and payload structs.
4. Define `Thread`, `Turn`, and `ThreadItem` API projection models.
5. Define `Op` and `EventMsg`.
6. Build an in-memory `ThreadStore`.
7. Build append-only JSONL history for thread items.
8. Add metadata/index projection for `thread/list` and `thread/read`.
9. Build a thread runtime that accepts `Op` and emits `EventMsg`.
10. Add a fake model adapter.
11. Build app-server request parsing and typed dispatch.
12. Implement `initialize`.
13. Implement `thread/start`.
14. Implement `turn/start`.
15. Stream notifications for turn and item lifecycle.
16. Persist emitted items through `LiveThread` or equivalent.
17. Add request serialization by thread id.
18. Add filesystem reads/writes with path policy.
19. Add command/process execution with lifecycle cleanup.
20. Add approval request/response handling.
21. Split execution into an exec-server process.
22. Add daemon lifecycle management for app-server.
23. Add extension systems: MCP, plugins, skills, apps/connectors.

Stop after step 17 for a first complete learning project. Steps 18-23 are
important, but they depend on the core request/event/storage architecture being
correct.

## Beginner Practice Milestones

Build these small milestones and test each one:

| Milestone | Passing condition |
|---|---|
| Envelope parser | Invalid JSON, unknown method, and malformed params return structured errors |
| Typed request mapper | `thread/start` and `turn/start` become typed enum variants |
| In-memory store | Creating a thread and appending items can be read back in order |
| JSONL store | Restarting the process can reconstruct thread history |
| Metadata projection | `thread/list` does not scan every item body |
| Runtime loop | One `Op::UserInput` produces turn-started, user-message, assistant-message, turn-complete events |
| App-server vertical slice | Client sends initialize, thread/start, turn/start and receives direct responses plus notifications |
| Serialization queue | Two writes to the same thread run in order; reads can run concurrently when safe |
| Fake tool call | Runtime emits a tool-call item, tool-output item, then final assistant item |
| Approval callback | Runtime pauses for client approval and resumes from the correlated response |

Each milestone should have tests before adding the next layer. This mirrors the
way the real system separates protocol, routing, runtime, storage, and
execution concerns.

## What To Ignore At First

Do not start with:

- Full terminal UI rendering.
- Plugin marketplace behavior.
- MCP server/resource discovery.
- Cloud task orchestration.
- Multi-OS remote execution.
- Sophisticated sandbox policy.
- Real-time voice or realtime transport features.
- Large tool catalogs.
- Search ranking or fuzzy file search.

Those pieces matter in the full Codex product, but they are not required to
learn the architecture. Add them only after the minimal thread-turn-event-store
loop works.

## Non-Negotiable Invariants

Keep these invariants in your implementation:

1. External wire messages are parsed once and converted into typed internal
   requests.
2. A thread is the unit of conversation, storage, and turn ordering.
3. A turn is not a single synchronous response; it is a lifecycle with events.
4. Direct responses acknowledge request handling; notifications stream runtime
   progress.
5. Canonical history is append-only.
6. Metadata/index data is a projection, not the source of truth.
7. Request serialization is scoped to the resource being mutated.
8. Execution is behind adapters, not embedded directly in UI code.
9. Background tasks have explicit ownership and cleanup.
10. Every long-running operation has correlation ids so responses and events
    can be routed correctly.

These invariants are the difference between a toy chatbot wrapper and a
working coding-agent runtime.

## Final Study Checklist

You have enough understanding to design this system when you can explain each
item without opening the code:

| Check | You should be able to answer |
|---|---|
| Product surfaces | Which user/integration surfaces exist and why they share core runtime |
| Crate/module split | Which module owns CLI, protocol, app-server, core, storage, execution, and extensions |
| Main models | What `Thread`, `Turn`, `ThreadItem`, `Op`, `EventMsg`, `RolloutItem`, and `ThreadStore` do |
| API shape | How `initialize`, `thread/start`, `turn/start`, `thread/read`, and `thread/list` work |
| Routing | How a method string becomes a typed handler call |
| Request serialization | Why writes to the same thread must be ordered |
| Runtime | How `Op` enters and `EventMsg` leaves |
| Storage | What goes to append-only history and what goes to metadata/index storage |
| UI surfacing | Why clients receive both direct responses and notifications |
| Processes | Which parts are OS processes and which are in-process tasks |
| Patterns | Which patterns are essential and which are optional refinements |
| Rebuild plan | Which 17 steps create the minimal working system |

When those answers are clear, move from reading to implementation using
`14-build-from-scratch-blueprint.md` as the build plan.

