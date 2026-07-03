# Layer 3 - Crate Responsibility Map

## Purpose Of This Layer

This layer answers: if you were rebuilding this system, what crates or modules
would you need, what each one owns, and how responsibility is split across the
workspace.

The key fact is that the Rust implementation is not one application crate. It
is a Cargo workspace with many small and medium crates listed in
`codex-rs/Cargo.toml:2`. The workspace member list runs from
`codex-rs/Cargo.toml:2` through `codex-rs/Cargo.toml:126`. The workspace
dependency table then gives each internal crate a package name and path, for
example `codex-core = { path = "core" }`, `codex-app-server = { path =
"app-server" }`, and `codex-tui = { path = "tui" }` in
`codex-rs/Cargo.toml`.

## Dependency Shape

The architecture is layered like this:

```text
User-facing entrypoints
  -> API facade crates
    -> core agent engine
      -> protocol/data types
      -> model/provider clients
      -> execution/sandboxing
      -> storage
      -> extensions/tools/plugins/MCP
      -> utilities
```

That is not a perfect tree. Some crates are shared by many layers. But the
design intent is still visible:

- `codex-cli` dispatches user commands and depends on many product crates.
- `codex-tui` and `codex-exec` are user-facing clients.
- `codex-app-server` exposes a JSON-RPC API for rich clients.
- `codex-core` owns the agent/session business logic.
- `codex-protocol` owns shared protocol types and should avoid business logic.
- `codex-api` and `codex-client` own HTTP/API transport to Codex/OpenAI
  services.
- `codex-thread-store`, `codex-rollout`, and `codex-state` own persistence.
- `codex-exec-server`, `codex-sandboxing`, and sandbox-specific crates own
  process/filesystem execution boundaries.

## Critical Path Crates

These are the crates you would implement first in a from-scratch version.

| Crate folder | Package | Responsibility | Evidence |
|---|---|---|---|
| `cli` | `codex-cli` | Top-level `codex` binary and command dispatcher. It depends on TUI, exec, app-server, daemon, MCP server, cloud tasks, login, config, core, and more. | `codex-rs/cli/Cargo.toml:1`, `codex-rs/cli/Cargo.toml:8`, `codex-rs/cli/Cargo.toml:18` |
| `tui` | `codex-tui` | Interactive terminal UI. It has a `codex-tui` binary and `codex_tui` library, and depends on app-server client/protocol, protocol, config, login, rollout, state, plugin, model crates, and UI libraries. | `codex-rs/tui/Cargo.toml:1`, `codex-rs/tui/Cargo.toml:8`, `codex-rs/tui/Cargo.toml:18` |
| `exec` | `codex-exec` | Non-interactive execution/review surface. It has a `codex-exec` binary and uses app-server client/protocol plus core/config/protocol. | `codex-rs/exec/Cargo.toml:1`, `codex-rs/exec/Cargo.toml:8`, `codex-rs/exec/Cargo.toml:23` |
| `app-server` | `codex-app-server` | Long-running app API server. Exposes `codex-app-server` binary and library. It depends on core, app-server protocol/transport, thread-store, state, exec-server, plugins, extensions, MCP, models, login, feedback, file search, file watcher, and more. | `codex-rs/app-server/Cargo.toml:1`, `codex-rs/app-server/Cargo.toml:7`, `codex-rs/app-server/Cargo.toml:24` |
| `app-server-protocol` | `codex-app-server-protocol` | Typed JSON-RPC API contract for app-server, including schema/TypeScript export dependencies. It depends on `codex-protocol`, `codex-shell-command`, path types, serde, schemars, `ts-rs`, and experimental API macros. | `codex-rs/app-server-protocol/Cargo.toml:1`, `codex-rs/app-server-protocol/Cargo.toml:13` |
| `app-server-client` | `codex-app-server-client` | Shared in-process app-server client used by `codex-exec` and `codex-tui`. It owns bootstrap, in-memory request/event transport, lifecycle, bounded queues, and graceful shutdown. | `codex-rs/app-server-client/README.md:1` |
| `app-server-transport` | `codex-app-server-transport` | Shared app-server transport layer. Used by app-server and client surfaces to separate message transport from request handling. | `codex-rs/app-server-transport/Cargo.toml:1` |
| `app-server-daemon` | `codex-app-server-daemon` | Unix-only lifecycle manager for detached app-server processes. Stores settings, pidfiles, and lock files under `CODEX_HOME/app-server-daemon`. | `codex-rs/app-server-daemon/README.md:3`, `codex-rs/app-server-daemon/README.md:106` |
| `core` | `codex-core` | Agent business logic: sessions, turns, tool orchestration, model context, approvals, model interaction, task execution, and most agent runtime behavior. | `codex-rs/core/README.md:1`, `codex-rs/core/Cargo.toml:1`, `codex-rs/core/Cargo.toml:17` |
| `protocol` | `codex-protocol` | Shared protocol/data types between core, TUI, and external app-server surfaces. It should have minimal dependencies and avoid material business logic. | `codex-rs/protocol/README.md:1` |
| `codex-api` | `codex-api` | Typed clients and request/response models for Responses, Compact, and memory summarize APIs. It is the wire-level API layer consumed by core. | `codex-rs/codex-api/README.md:1` |
| `codex-client` | `codex-client` | Generic HTTP/websocket client substrate beneath `codex-api` and other networked clients. | `codex-rs/codex-client/Cargo.toml:1` |
| `exec-server` | `codex-exec-server` | JSON-RPC server/library for process, filesystem, and HTTP operations, local websocket mode, and remote relay mode. | `codex-rs/exec-server/README.md:1`, `codex-rs/exec-server/Cargo.toml:1` |
| `exec-server-protocol` | `codex-exec-server-protocol` | Wire protocol constants and payload types for exec-server methods such as `process/start`, `process/read`, `fs/readFile`, and `http/request`. | `codex-rs/exec-server-protocol/Cargo.toml:1` |
| `thread-store` | `codex-thread-store` | Thread storage boundary. Defines `ThreadStore`, local and in-memory implementations, append-only history, metadata updates, and active `LiveThread` persistence. | `codex-rs/thread-store/README.md:1`, `codex-rs/thread-store/Cargo.toml:1` |
| `rollout` | `codex-rollout` | Durable rollout/session history recording and loading. Used by thread-store, core, TUI, app-server, CLI, and exec. | `codex-rs/rollout/Cargo.toml:1` |
| `state` | `codex-state` | SQLite-backed state runtime and migrations for state/logs/goals/memories. Used by core, app-server, TUI, thread-store, and CLI. | `codex-rs/state/Cargo.toml:1` |
| `config` | `codex-config` | Config loading and config model types. Pulls together exec policy, features, filesystem, model provider info, network proxy settings, path handling, TOML, and schemas. | `codex-rs/config/Cargo.toml:1`, `codex-rs/config/Cargo.toml:12` |

## User-Facing Entry Point Crates

| Crate folder | Package | What it owns |
|---|---|---|
| `cli` | `codex-cli` | Main `codex` executable. This is the command router, not the whole product. It calls into TUI, exec, app-server, daemon, MCP server, login, cloud task, plugin, and maintenance crates. |
| `tui` | `codex-tui` | Interactive terminal experience. Depends on app-server client/protocol for runtime communication, and on ratatui/crossterm for terminal UI rendering. |
| `exec` | `codex-exec` | Non-interactive agent runs and review runs. Its source explicitly states stdout is reserved for final output unless JSONL mode is used. |
| `cloud-tasks` | `codex-cloud-tasks` | CLI surface for creating and managing cloud tasks through `codex-cloud-tasks-client`, login, config, and model selection. |
| `mcp-server` | `codex-mcp-server` | Codex exposed as an MCP server. Has its own binary and library. Uses core, exec-server, protocol, MCP crates, and approval handling modules. |
| `app-server` | `codex-app-server` | Rich-client API server binary and library. Also includes a test notification capture binary. |
| `app-server-daemon` | `codex-app-server-daemon` | Detached app-server lifecycle operations used by CLI daemon commands. |
| `execpolicy` | `codex-execpolicy` | Execution policy binary/library. |
| `execpolicy-legacy` | `codex-execpolicy-legacy` | Legacy execution policy binary/library. |
| `linux-sandbox` | `codex-linux-sandbox` | Linux sandbox helper binary/library. |
| `windows-sandbox-rs` | `codex-windows-sandbox` | Windows sandbox setup and command runner binaries plus library. |
| `file-search` | `codex-file-search` | File search binary/library. |
| `responses-api-proxy` | `codex-responses-api-proxy` | Responses API proxy binary/library. |
| `stdio-to-uds` | `codex-stdio-to-uds` | Utility binary/library for bridging stdio to Unix domain sockets. |
| `code-mode-host` | `codex-code-mode-host` | Host binary/library for code-mode. |
| `bwrap` | `codex-bwrap` | Bubblewrap wrapper binary. |
| `apply-patch` | `codex-apply-patch` | Patch application library and binary. |

## Core Engine And Agent Runtime Crates

| Crate folder | Package | What it owns |
|---|---|---|
| `core` | `codex-core` | Central agent/session runtime. It depends on model APIs, protocol, tools, MCP, config, sandboxing, thread-store, rollout, state, plugins, memories, file system, shell command, model provider, feedback, and network proxy. This is the heaviest crate. |
| `core-api` | `codex-core-api` | Shared API types/interfaces adjacent to core. |
| `protocol` | `codex-protocol` | Shared internal and external protocol types. README explicitly says this crate should avoid material business logic. |
| `context-fragments` | `codex-context-fragments` | Context fragment support used by core and extension APIs. |
| `message-history` | `codex-message-history` | Message history support used by TUI. |
| `prompts` | `codex-prompts` | Prompt assets and prompt construction support. |
| `response-debug-context` | `codex-response-debug-context` | Debug context around model responses. |
| `collaboration-mode-templates` | `codex-collaboration-mode-templates` | Collaboration-mode prompt/template material. |
| `agent-identity` | `codex-agent-identity` | Agent identity concepts. |
| `agent-graph-store` | `codex-agent-graph-store` | Agent graph persistence/support used by core. |
| `features` | `codex-features` | Feature flags or feature configuration shared across runtime crates. |
| `feedback` | `codex-feedback` | Feedback capture/reporting support used by CLI, TUI, app-server, and core-adjacent surfaces. |
| `hooks` | `codex-hooks` | Hook support used by app-server and core. |
| `install-context` | `codex-install-context` | Install/runtime context used by config, TUI, CLI, core, and thread-store. |

## API, Model, Auth, And Cloud Crates

| Crate folder | Package | What it owns |
|---|---|---|
| `codex-api` | `codex-api` | Typed API clients and request/response models for Responses, Compact, and memory summarize APIs. |
| `codex-client` | `codex-client` | Generic client transport foundation under API clients. |
| `backend-client` | `codex-backend-client` | Backend service client used by app-server. |
| `codex-backend-openapi-models` | `codex-backend-openapi-models` | Generated or shared OpenAPI model definitions for backend APIs. |
| `model-provider` | `codex-model-provider` | Model provider abstraction/configuration layer. |
| `model-provider-info` | `codex-model-provider-info` | Model/provider metadata. |
| `models-manager` | `codex-models-manager` | Model listing/management logic used by CLI, TUI, app-server, and core. |
| `login` | `codex-login` | Login/auth flows. |
| `keyring-store` | `codex-keyring-store` | OS keyring persistence for credentials/secrets. |
| `secrets` | `codex-secrets` | Secret handling. |
| `aws-auth` | `codex-aws-auth` | AWS authentication helper support. |
| `cloud-config` | `codex-cloud-config` | Cloud configuration. |
| `cloud-tasks-client` | `codex-cloud-tasks-client` | Client for cloud task backend APIs. |
| `cloud-tasks-mock-client` | `codex-cloud-tasks-mock-client` | Mock cloud task client for tests/dev. |
| `cloud-tasks` | `codex-cloud-tasks` | CLI cloud task feature layer. |
| `chatgpt` | `codex-chatgpt` | ChatGPT-specific integration support. |
| `lmstudio` | `codex-lmstudio` | LM Studio integration. |
| `ollama` | `codex-ollama` | Ollama integration. |
| `realtime-webrtc` | `codex-realtime-webrtc` | Realtime/WebRTC integration support. |

## Server And Protocol Boundary Crates

| Crate folder | Package | What it owns |
|---|---|---|
| `app-server` | `codex-app-server` | App-facing JSON-RPC server runtime, request dispatch, process loops, thread/turn APIs, and extension/plugin surfaces. |
| `app-server-protocol` | `codex-app-server-protocol` | App-server request/response/notification types and schema export. |
| `app-server-client` | `codex-app-server-client` | Typed in-process client facade for TUI and exec. |
| `app-server-transport` | `codex-app-server-transport` | Transport abstraction for app-server messages. |
| `app-server-test-client` | `codex-app-server-test-client` | Test client for app-server integration tests. |
| `app-server/tests/common` | `app_test_support` | Shared app-server test support crate. |
| `exec-server` | `codex-exec-server` | Process/filesystem/HTTP RPC server. Owns transport, protocol wiring, and handlers. |
| `exec-server-protocol` | `codex-exec-server-protocol` | Exec-server method constants and payload types. |
| `mcp-server` | `codex-mcp-server` | Codex as MCP server. |
| `mcp-server/tests/common` | `mcp_test_support` | MCP server test support crate. |
| `codex-mcp` | `codex-mcp` | MCP client-side catalog, connection management, tool aggregation, and MCP server integration for core/app-server surfaces. |
| `rmcp-client` | `codex-rmcp-client` | RMCP client lifecycle support used by MCP integration. |
| `uds` | `codex-uds` | Unix domain socket utilities. |

## Persistence Crates

| Crate folder | Package | What it owns |
|---|---|---|
| `thread-store` | `codex-thread-store` | Thread storage abstraction, local/in-memory implementations, metadata update API, append-only history via rollout, and `LiveThread`. |
| `rollout` | `codex-rollout` | Rollout JSONL recording/loading support. |
| `rollout-trace` | `codex-rollout-trace` | Rollout tracing support. |
| `state` | `codex-state` | SQLite state runtime and migrations. |
| `external-agent-sessions` | `codex-external-agent-sessions` | External agent session storage/support. |
| `external-agent-migration` | `codex-external-agent-migration` | Migration support for external agent sessions/data. |
| `memories/read` | `codex-memories-read` | Memory read support. |
| `memories/write` | `codex-memories-write` | Memory write support. |
| `memories/README.md` | n/a | Documents memory subsystem layout. |

## Execution, Filesystem, Sandbox, And Policy Crates

| Crate folder | Package | What it owns |
|---|---|---|
| `exec-server` | `codex-exec-server` | Remote/local execution RPC server for processes, filesystem, and HTTP. |
| `utils/pty` | `codex-utils-pty` | PTY handling used by exec-server and app-server. |
| `file-system` | `codex-file-system` | Filesystem abstractions used by core, exec-server, and config. |
| `file-search` | `codex-file-search` | File search implementation and binary. |
| `file-watcher` | `codex-file-watcher` | File watching used by app-server. |
| `git-utils` | `codex-git-utils` | Git helpers used broadly by CLI, TUI, core, config, app-server, and thread-store. |
| `sandboxing` | `codex-sandboxing` | Cross-platform sandboxing interface. |
| `linux-sandbox` | `codex-linux-sandbox` | Linux sandbox implementation/helper. |
| `windows-sandbox-rs` | `codex-windows-sandbox` | Windows sandbox implementation and helper binaries. |
| `process-hardening` | `codex-process-hardening` | Process-hardening support. |
| `shell-command` | `codex-shell-command` | Shell command modeling/parsing/support. |
| `shell-escalation` | `codex-shell-escalation` | Unix shell escalation wrapper support. |
| `execpolicy` | `codex-execpolicy` | Current execution policy. |
| `execpolicy-legacy` | `codex-execpolicy-legacy` | Legacy execution policy. |
| `network-proxy` | `codex-network-proxy` | Local network policy enforcement proxy. README says it runs HTTP and SOCKS5 proxies and enforces allow/deny plus limited mode. |

## Extension, Plugin, Tool, Skill, And App Crates

| Crate folder | Package | What it owns |
|---|---|---|
| `ext/extension-api` | `codex-extension-api` | Shared extension-facing API. Depends on config, context fragments, protocol, tools, and path types. |
| `tools` | `codex-tools` | Shared support for model-visible tools outside core: tool specs, discovery models, adapters, executable-tool contracts. |
| `plugin` | `codex-plugin` | Plugin manifest/path/metadata support. |
| `core-plugins` | `codex-core-plugins` | Built-in/core plugin support. |
| `skills` | `codex-skills` | Skill data/model support. |
| `core-skills` | `codex-core-skills` | Built-in/core skill support. |
| `ext/skills` | `codex-skills-extension` | Skills extension implementation. |
| `ext/mcp` | `codex-mcp-extension` | MCP extension implementation. |
| `ext/connectors` | `codex-connectors-extension` | Connectors extension implementation. |
| `connectors` | `codex-connectors` | Connector models/support. |
| `ext/goal` | `codex-goal-extension` | Goal extension implementation. |
| `ext/guardian` | `codex-guardian` | Guardian extension implementation. |
| `ext/image-generation` | `codex-image-generation-extension` | Image generation extension implementation. |
| `ext/memories` | `codex-memories-extension` | Memories extension implementation. |
| `ext/web-search` | `codex-web-search-extension` | Web search extension implementation. |
| `code-mode` | `codex-code-mode` | Code-mode support. |
| `code-mode-protocol` | `codex-code-mode-protocol` | Code-mode protocol types. |
| `code-mode-host` | `codex-code-mode-host` | Code-mode host process/library. |

## Utility And Support Crates

These are not product surfaces by themselves. They keep repeated mechanics out
of larger crates.

| Crate folder | Package | What it owns |
|---|---|---|
| `analytics` | `codex-analytics` | Analytics event support. |
| `ansi-escape` | `codex-ansi-escape` | ANSI escape handling. |
| `arg0` | `codex-arg0` | argv[0]/binary-name helper logic. |
| `async-utils` | `codex-async-utils` | Async helper utilities. |
| `codex-home` | `codex-home` | Codex home directory resolution. |
| `otel` | `codex-otel` | OpenTelemetry support. |
| `terminal-detection` | `codex-terminal-detection` | Terminal detection logic. |
| `test-binary-support` | `codex-test-binary-support` | Test support for locating/spawning binaries. |
| `thread-manager-sample` | `codex-thread-manager-sample` | Sample thread manager. |
| `v8-poc` | `codex-v8-poc` | V8 proof-of-concept crate. |
| `codex-experimental-api-macros` | `codex-experimental-api-macros` | Macros for experimental API annotation/gating. |
| `utils/absolute-path` | `codex-utils-absolute-path` | Absolute path type/helpers. |
| `utils/path-uri` | `codex-utils-path-uri` | Path URI conversion/helpers. |
| `utils/cargo-bin` | `codex-utils-cargo-bin` | Stable binary lookup for Cargo/Bazel tests. |
| `utils/cache` | `codex-utils-cache` | Cache helpers. |
| `utils/image` | `codex-utils-image` | Image helpers. |
| `utils/json-to-toml` | `codex-utils-json-to-toml` | JSON-to-TOML conversion. |
| `utils/home-dir` | `codex-utils-home-dir` | Home directory helper. |
| `utils/readiness` | `codex-utils-readiness` | Readiness signaling/helpers. |
| `utils/rustls-provider` | `codex-utils-rustls-provider` | Rustls provider setup. |
| `utils/string` | `codex-utils-string` | String helpers. |
| `utils/cli` | `codex-utils-cli` | CLI helpers. |
| `utils/elapsed` | `codex-utils-elapsed` | Elapsed time formatting/helpers. |
| `utils/sandbox-summary` | `codex-utils-sandbox-summary` | Sandbox summary formatting/support. |
| `utils/sleep-inhibitor` | `codex-utils-sleep-inhibitor` | Sleep inhibition helpers. |
| `utils/approval-presets` | `codex-utils-approval-presets` | Approval preset helpers. |
| `utils/oss` | `codex-utils-oss` | Open-source/environment helpers. |
| `utils/output-truncation` | `codex-utils-output-truncation` | Output truncation support. |
| `utils/path-utils` | `codex-utils-path` | Path utility functions/types. |
| `utils/plugins` | `codex-utils-plugins` | Plugin path/cache/utility helpers. |
| `utils/fuzzy-match` | `codex-utils-fuzzy-match` | Fuzzy matching helpers. |
| `utils/stream-parser` | `codex-utils-stream-parser` | Streaming parser utilities. |
| `utils/template` | `codex-utils-template` | Template rendering helpers. |

## Test Support Crates

| Crate folder | Package | What it owns |
|---|---|---|
| `core/tests/common` | `core_test_support` | Shared integration test support for `codex-core`. |
| `app-server/tests/common` | `app_test_support` | Shared integration test support for app-server tests. |
| `mcp-server/tests/common` | `mcp_test_support` | Shared integration test support for MCP server tests. |
| `cloud-tasks-mock-client` | `codex-cloud-tasks-mock-client` | Mock cloud task client. |
| `app-server-test-client` | `codex-app-server-test-client` | App-server test client. |
| `test-binary-support` | `codex-test-binary-support` | Binary lookup/spawning support for tests. |

## What This Means For A From-Scratch Design

Do not start by creating every crate above. Start with a minimal version of the
same boundaries:

1. `protocol`
   - Define stable domain and wire types first.
   - Keep business logic out of this layer.

2. `config`
   - Load product configuration, auth choices, sandbox policy, model choices,
     and workspace roots.

3. `api-client`
   - Own HTTP/SSE/websocket details for model/backend calls.
   - Return typed events, not raw JSON strings, to the engine.

4. `core`
   - Own session state, turn orchestration, model request construction, tool
     calls, approvals, cancellation, and event emission.

5. `storage`
   - Own append-only conversation history plus queryable metadata.
   - Keep one write API for history and one write API for metadata.

6. `exec`
   - Own process execution, filesystem reads/writes, sandboxing, and remote
     execution boundaries.

7. `app-api`
   - Expose rich-client JSON-RPC methods over transport-agnostic channels.

8. `ui-cli`
   - Build interactive and non-interactive clients on top of the app API or
     core API.

9. `extensions`
   - Add tools, plugins, skills, MCP, connectors, memories, web search, and
     image generation only after the core loop is stable.

10. `utilities`
    - Extract utility crates only when two or more higher layers need the same
      behavior.

## Boundary Rules To Preserve

- Keep `protocol` as type definitions and serialization contracts.
- Keep `api-client` responsible for wire transport, retries, headers, and
  stream parsing.
- Keep `core` responsible for agent decisions and turn orchestration.
- Keep `app-server` responsible for client/server request routing, not model
  reasoning itself.
- Keep `exec-server` responsible for process/filesystem/HTTP execution
  mechanics, not UI policy.
- Keep `thread-store` responsible for storage mutation boundaries.
- Keep extension/plugin crates outside core unless they must participate in the
  central turn loop.
- Keep user-facing binaries thin: parse command, load config, choose a surface,
  call into libraries.

## Layer 3 Completion Criteria

This layer is complete because it identifies:

- the workspace membership source,
- the critical path crates,
- the user-facing entrypoint crates,
- the core engine crates,
- API/model/auth/cloud crates,
- server/protocol boundary crates,
- persistence crates,
- execution/sandboxing crates,
- extension/plugin/tool/skill crates,
- utility/test crates,
- and the minimal from-scratch crate sequence.
