# Codex Codebase Learning Index

This folder contains learning artifacts for understanding the Codex codebase
well enough to design and implement a similar system from scratch.

## Artifact Boundary

All artifacts for this study live under `docs/learnings/`. The study reads the
codebase, but it must not create, edit, or delete code or documentation outside
this directory.

## Reading Order

0. [Progress Tracker](progress-tracker.md)
1. [01 Repository Orientation](01-repository-orientation.md)
2. [04 Product Surface Inventory](04-product-surface-inventory.md)
3. [05 Crate Responsibility Map](05-crate-responsibility-map.md)
4. [06 Primary Use Cases](06-primary-use-cases.md)
5. [07 Core Data Models](07-core-data-models.md)
6. [08 API and Protocol Catalog](08-api-and-protocol-catalog.md)
7. [09 End-to-End Request Flows](09-end-to-end-request-flows.md)
8. [10 Routing and Handler Mechanics](10-routing-and-handler-mechanics.md)
9. [11 Storage and Update Paths](11-storage-and-update-paths.md)
10. [02 Runtime Processes and Task Topology](02-runtime-processes-and-task-topology.md)
11. [12 System Design Synthesis](12-system-design-synthesis.md)
12. [13 Design Patterns](13-design-patterns.md)
13. [14 Build From Scratch Blueprint](14-build-from-scratch-blueprint.md)
14. [15 Final Learning Guide](15-final-learning-guide.md)
15. [03 API and Request Routing](03-api-and-request-routing.md)

## Current Findings Snapshot

- The top-level product is a local coding agent exposed through several
  surfaces: CLI/TUI, app-server, app-server daemon, exec-server, MCP server,
  SDKs, plugins, skills, and storage crates.
- The primary product surfaces are now inventoried with concrete entry points:
  interactive terminal, non-interactive exec/review, app-server, daemon,
  exec-server, MCP, plugins, skills, apps, cloud tasks, SDKs, and packaging.
- The main Rust implementation lives in `codex-rs/`, a Cargo workspace with
  many `codex-*` crates.
- The crate map is now explicit: thin user-facing binaries call into API
  facades, which call into `codex-core`, shared protocols, model/API clients,
  storage crates, execution/sandboxing crates, and extension/plugin/tool
  crates.
- Primary use cases are now mapped to concrete commands, app-server methods,
  protocol payload structs, handlers, storage/update paths, and surfaced
  responses/notifications.
- Core data models are now mapped across API envelopes, Thread/Turn/ThreadItem
  projections, ThreadStart/TurnStart params, UserInput, Op/Event/EventMsg,
  ResponseItem, RolloutItem/RolloutLine, ThreadStore/LiveThread, local storage,
  exec-server payloads, and tool schema/call/output types.
- API and protocol surfaces are now cataloged across app-server JSON-RPC,
  app-server typed in-process channels, core Op/Event/EventMsg, exec-server
  JSON-RPC, server-to-client approval/tool requests, notifications, SDK JSONL,
  and storage trait boundaries.
- End-to-end request flows are now mapped from clients through transports,
  app-server dispatch, domain handlers, core `Op` submission, event streams,
  storage writes, direct responses, notifications, and UI/read projections.
- Routing and handler mechanics are now mapped across CLI subcommands,
  transport events, app-server processor and outbound loops, typed request
  dispatch, request serialization queues, domain processors, outgoing
  responses/errors/notifications, server-to-client callback routing,
  connection cleanup, in-process app-server routing, core `Op` dispatch, task
  routing, and exec-server RPC registration.
- Storage and update paths are now mapped across the `ThreadStore` boundary,
  `LiveThread`, rollout JSONL history, SQLite metadata/index storage, live
  append ordering, explicit and derived metadata updates, archive/delete paths,
  read/list/turn/item projections, live UI notifications, config/auth/daemon
  side stores, skills/plugins, goals, memories, logs, and from-scratch storage
  implementation steps.
- `codex app-server` is a long-running JSON-RPC server process for rich clients.
- `codex app-server daemon` is a Unix-only lifecycle manager that starts and
  controls app-server as a detached pidfile-backed process.
- `codex exec-server` is a separate JSON-RPC process for filesystem/process
  operations, including remote execution environments.
- Inside a loaded Codex session, work is driven by in-process Tokio tasks such
  as `SessionTask` implementations.
- The overall system design is now synthesized as a typed, event-driven,
  thread-centered agent runtime: requests enter through surfaces, become typed
  payloads, submit `Op` values into core sessions, emit `EventMsg` values,
  persist canonical history plus metadata, and surface progress through direct
  responses and notifications.
- The high-level and low-level design patterns are now mapped: thin entrypoint
  routing, typed protocol boundaries, JSON envelope conversion, initialization
  and capability gates, resource-scoped serialization, domain processors,
  response plus notification streaming, thread/session facades, command/event
  splitting, composition roots, port/adapter storage, append-only history plus
  projections, background workers, router registries, transport substitution,
  correlation IDs, backpressure, cleanup, and structured error envelopes.
- The build-from-scratch blueprint is now concrete: it defines the target
  system, non-negotiable invariants, workspace/module skeleton, protocol
  envelope, public data models, storage port, append-only history, core
  Op/Event runtime, app-server transport and routing, thread/turn processors,
  request serialization, model adapter, filesystem/process tools, approval
  flow, exec-server split, daemon, extensions, minimal vertical slice, exact
  implementation order, beginner test plan, common failure modes, and
  definition of done.
- The final learning guide now ties the study together into a beginner-oriented
  source-backed reading path, one-page mental model, original-question mapping,
  core runtime flow, minimum data/API/storage/runtime sets, routing rules,
  process/task distinctions, design-pattern sequence, exact build order,
  practice milestones, scope exclusions, invariants, and final study checklist.

## Remaining Planned Artifacts

- None. All planned learning artifacts are complete.
