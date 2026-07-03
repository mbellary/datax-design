# Codex Architecture Study Progress Tracker

This tracker maps directly to `architecture-study-execution-plan.md`.

## Working Rule

- All artifacts stay under `docs/learnings/`.
- Work proceeds one layer at a time.
- A layer is marked complete only when it has a dedicated artifact or is fully
  covered by an existing artifact with source-backed evidence.
- No handwaving: each completed layer must cite concrete files, functions,
  structs, modules, methods, or request paths.

## Current Layer

Next layer to execute: **None. All planned layers are complete.**

Reason: Layer 14 is now complete in `15-final-learning-guide.md`. The study has
completed every layer in `architecture-study-execution-plan.md`.

## Layer Status

| Layer | Plan Section | Status | Artifact |
|---:|---|---|---|
| 1 | Repository orientation | Complete | `01-repository-orientation.md` |
| 2 | Product surface inventory | Complete | `04-product-surface-inventory.md` |
| 3 | Crate responsibility map | Complete | `05-crate-responsibility-map.md` |
| 4 | Primary use cases | Complete | `06-primary-use-cases.md` |
| 5 | Core data models | Complete | `07-core-data-models.md` |
| 6 | API and protocol catalog | Complete | `08-api-and-protocol-catalog.md` |
| 7 | End-to-end request flows | Complete | `09-end-to-end-request-flows.md` |
| 8 | Routing and handler mechanics | Complete | `10-routing-and-handler-mechanics.md` |
| 9 | Storage and update paths | Complete | `11-storage-and-update-paths.md` |
| 10 | Runtime processes and task topology | Complete | `02-runtime-processes-and-task-topology.md` |
| 11 | System design synthesis | Complete | `12-system-design-synthesis.md` |
| 12 | Design patterns | Complete | `13-design-patterns.md` |
| 13 | Build-from-scratch blueprint | Complete | `14-build-from-scratch-blueprint.md` |
| 14 | Final learning guide | Complete | `15-final-learning-guide.md` |

## Completed So Far

1. Created the execution plan:
   - `architecture-study-execution-plan.md`

2. Created the study index:
   - `00-index.md`

3. Completed Layer 1, repository orientation:
   - `01-repository-orientation.md`
   - Covers the repository purpose, top-level layout, Rust workspace structure,
     crate groupings, and CLI entry-point orientation.

4. Completed Layer 10, runtime processes and task topology:
   - `02-runtime-processes-and-task-topology.md`
   - Covers OS processes, daemon-managed processes, server loops, in-process
     Tokio tasks, exec-server process topology, session tasks, and storage
     persistence boundaries.

5. Started Layers 6, 8, and 9:
   - `03-api-and-request-routing.md`
   - Covers app-server JSON-RPC routing, exec-server RPC routing, selected
     dispatch mechanics, storage request paths, and major payload categories.

6. Completed Layer 2, product surface inventory:
   - `04-product-surface-inventory.md`
   - Covers terminal, CLI, app-server, daemon, exec-server, MCP, plugins,
     skills, apps/connectors, cloud tasks, SDKs, packaging, and maintenance
     surfaces with source-backed entry points.

7. Completed Layer 3, crate responsibility map:
   - `05-crate-responsibility-map.md`
   - Covers the Cargo workspace shape, critical path crates, entrypoint crates,
     core/runtime crates, API/model/cloud crates, server/protocol crates,
     persistence crates, execution/sandboxing crates, extension/plugin/tool
     crates, utilities, test support, and a from-scratch crate sequence.

8. Completed Layer 4, primary use cases:
   - `06-primary-use-cases.md`
   - Covers interactive local sessions, non-interactive exec, review, rich
     client conversations, stored-thread read/resume flows, lifecycle metadata,
     utility command execution, filesystem APIs, exec-server, app-server daemon,
     skills/plugins/apps, MCP calls, auth/account, cloud tasks, SDK embedding,
     request/response shapes, and the minimal implementation order.

9. Completed Layer 5, core data models:
   - `07-core-data-models.md`
   - Covers app-server RPC envelopes, Thread/Turn/ThreadItem API models,
     ThreadStart/TurnStart payloads, UserInput, core Op/Event/EventMsg,
     ResponseItem, RolloutItem/RolloutLine, ThreadStore/LiveThread,
     local JSONL plus SQLite storage, exec-server process/filesystem payloads,
     tool schema/call/output models, and the canonical request-to-data flow.

10. Completed Layer 6, API and protocol catalog:
    - `08-api-and-protocol-catalog.md`
    - Covers app-server JSON-RPC envelopes, typed ClientRequest and
      ServerNotification protocol, method strings, Params/Response payload
      types, dispatcher ownership, server-to-client request methods, core
      Op/Event/EventMsg protocol, exec-server JSON-RPC methods and payloads,
      SDK JSONL boundary, storage trait boundary, routing mechanics, and the
      from-scratch implementation sequence for the API layer.

11. Completed Layer 7, end-to-end request flows:
    - `09-end-to-end-request-flows.md`
    - Covers rich-client thread start, turn start, TUI in-process routing,
      non-interactive exec, review, interrupt, stored read/list/page flows,
      metadata updates, filesystem utilities, command/process utilities,
      exec-server remote runtime, MCP, plugins/skills/apps, account, remote
      control/daemon, SDK streaming, core event projection, storage flow, and
      the from-scratch implementation order.

12. Completed Layer 8, routing and handler mechanics:
    - `10-routing-and-handler-mechanics.md`
    - Covers CLI command routing, app-server transport events, app-server
      processor/outbound loops, JSON-RPC-like request deserialization,
      `ClientRequest` dispatch, initialization gating, request serialization
      queues, domain processor ownership, outgoing response/error/notification
      routing, server-to-client request callbacks, connection cleanup,
      in-process routing, core `Op` dispatch, core task routing, exec-server
      RPC routing, error routing, correlation keys, and a from-scratch routing
      implementation sequence.

13. Completed Layer 9, storage and update paths:
    - `11-storage-and-update-paths.md`
    - Covers the `ThreadStore` boundary, `LiveThread` write-through behavior,
      local rollout JSONL history, SQLite metadata/index storage, turn append
      ordering, explicit and derived metadata updates, archive/unarchive/delete
      paths, read/list/turn/item projections, live UI notifications, config,
      auth, daemon, skills, plugin, memory, goal, and log side stores, the
      consistency model, request-to-storage mappings, and a from-scratch
      implementation sequence.

14. Completed Layer 11, system design synthesis:
    - `12-system-design-synthesis.md`
    - Covers the overall system shape, major components, data/control planes,
      runtime modes, core invariants, turn flow, storage surfacing, a
      from-scratch implementation order, minimal data/API/storage sets, routing
      strategy, common beginner mistakes, and a map back to the original study
      questions.

15. Completed Layer 12, design patterns:
    - `13-design-patterns.md`
    - Covers thin entrypoint routing, typed protocol boundaries, JSON envelopes
      with typed internal requests, initialization and capability gates,
      resource-scoped serialization, domain processor ownership, response plus
      notification streams, thread/session facades, command/event splitting,
      composition roots, port/adapter storage, append-only history plus
      projections, background workers, router registries, transport
      substitution, low-level correlation/backpressure/cleanup/error patterns,
      the pattern composition in `turn/start`, anti-patterns, and a
      reimplementation module shape.

16. Completed Layer 13, build-from-scratch blueprint:
    - `14-build-from-scratch-blueprint.md`
    - Covers a concrete implementation sequence for a Codex-like system:
      workspace/module skeleton, protocol envelope, typed requests, public
      Thread/Turn/Item models, storage trait, append-only rollout history,
      core Op/Event runtime, app-server transport and routing, thread/turn
      processors, request serialization, model-provider adapter,
      filesystem/process tools, approval flow, exec-server split, daemon,
      extension systems, a minimal vertical slice, exact phase order,
      beginner test plan, failure modes, and definition of done.

17. Completed Layer 14, final learning guide:
    - `15-final-learning-guide.md`
    - Ties the full study together into a beginner-oriented reading path,
      source-anchor map, mental model, original-question mapping, core runtime
      flow, data/API/storage/runtime minimums, routing rules, process/task
      distinctions, design-pattern sequence, exact build order, beginner
      practice milestones, scope exclusions, invariants, and final study
      checklist.

## Next Execution Order

All planned layers are complete.

Layers 1 through 14 are complete and will not be expanded unless the user asks
for a specific follow-up artifact.

## Status Legend

- Not started: no dedicated evidence-backed write-up exists yet.
- Partial: evidence-backed content exists, but the layer still needs a complete
  dedicated treatment or consolidation.
- Complete: the layer has enough concrete source-backed detail to teach and
  reimplement that part of the system.
