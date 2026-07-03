# Codex Architecture Study Execution Plan

This document defines the order for studying the Codex codebase and producing
learning artifacts. All artifacts for this effort must be created under
`docs/learnings/`.

## Execution Order

1. Repository orientation
   - Map the top-level folders, important READMEs, workspace layout, and major
     entry points.

2. Product surface inventory
   - Identify the user-facing and integration-facing surfaces: CLI, TUI,
     app-server, exec-server, MCP, plugins, skills, SDKs, and storage.

3. Crate responsibility map
   - Group crates by role: UI, core agent logic, protocol, persistence,
     execution, sandboxing, model/backend access, app APIs, extensions, and
     utilities.

4. Primary use cases
   - Describe the main workflows the system supports, including interactive
     coding, tool execution, approvals, file edits, PR/code-review workflows,
     app integration, remote execution, plugins, and skills.

5. Core data models
   - Extract and explain the important structs and enums for threads, sessions,
     messages, events, tool calls, approvals, configuration, sandbox policy,
     stored state, and request/response payloads.

6. API and protocol catalog
   - Document the major system boundaries: app-server JSON-RPC methods,
     exec-server protocol, core protocol events, OpenAI Responses API payloads,
     MCP interactions, CLI commands, and SDK APIs.

7. End-to-end request flows
   - Trace representative flows from user input through model requests, tool
     calls, execution, approvals, file updates, UI updates, and persistence.

8. Routing and handler mechanics
   - Explain dispatch layers: CLI command routing, TUI event handling,
     app-server method dispatch, core operation handling, model event handling,
     and tool-call routing.

9. Storage and update paths
   - Track where data is stored and mutated: config, Codex home, thread store,
     rollout/message history, keyring/secrets, in-memory state, UI state, and
     generated artifacts.

10. Runtime processes and task topology
    - Explicitly distinguish operating-system processes, daemon-managed
      lifecycle processes, server loops, in-process async tasks, and storage
      workers.

11. System design synthesis
    - Produce architecture diagrams and maps: component map, sequence diagrams,
      storage map, API boundary map, and UI surfacing map.

12. Design patterns
    - Identify high-level and low-level patterns such as event loops,
      actor-like async tasks, typed protocol DTOs, adapters, repositories,
      command handlers, capability-based tools, and sandbox policy enforcement.

13. Build-from-scratch blueprint
    - Convert the findings into a staged implementation roadmap for someone
      building a similar system with no prior experience.

14. Final learning guide
    - Assemble the findings into a teaching-oriented document that explains
      what the system is, why it exists, how it works, where the code lives, how
      requests flow, and how to recreate it.

## Artifact Rule

No files outside `docs/learnings/` should be created, edited, or deleted for
this learning effort.
