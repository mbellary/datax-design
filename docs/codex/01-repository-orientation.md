# Repository Orientation

## What This System Is

This repository implements Codex CLI, a local coding agent. The root README
states that Codex CLI runs locally on a user's computer and can be used from the
terminal, editor integrations, desktop app flows, and cloud/remote surfaces.

The implementation is not one monolithic binary. It is a multi-surface system:

- an interactive CLI/TUI
- a non-interactive `exec` mode
- an app-server JSON-RPC server for richer clients
- an app-server daemon for remote-management lifecycle
- an exec-server for process/filesystem execution
- an MCP server
- SDKs
- plugin and skill infrastructure
- local persistence for threads, rollout history, config, state, goals, memory,
  logs, and credentials

## Top-Level Layout

Important top-level directories:

- `codex-rs/`: main Rust workspace.
- `codex-cli/`: npm/package wrapper and scripts for CLI distribution.
- `sdk/typescript/`: TypeScript SDK.
- `sdk/python/`: Python SDK.
- `docs/`: repository contribution/install docs and this learning folder.
- `tools/`: repository tools such as the Rust argument-comment lint.
- `scripts/`: packaging/install helper scripts.
- `third_party/`: vendored or pinned third-party assets.

## Rust Workspace

The Rust workspace is declared in `codex-rs/Cargo.toml`. The workspace contains
many crates, and crate package names are prefixed with `codex-`.

Core crates by responsibility:

| Responsibility | Crates |
| --- | --- |
| CLI entry point and command routing | `cli` |
| Terminal UI | `tui` |
| Core agent/session logic | `core`, `core-api`, `protocol` |
| Rich-client API server | `app-server`, `app-server-protocol`, `app-server-transport`, `app-server-client`, `app-server-daemon` |
| Execution server | `exec-server`, `exec-server-protocol`, `exec`, `execpolicy`, `sandboxing` |
| Persistence | `thread-store`, `rollout`, `state`, `codex-home`, `message-history`, `keyring-store` |
| Model/backend access | `backend-client`, `model-provider`, `model-provider-info`, `models-manager`, `codex-api` |
| Extensions | `plugin`, `skills`, `core-plugins`, `core-skills`, `ext/*`, `mcp-server`, `codex-mcp` |
| Utilities | `utils/*`, `file-system`, `git-utils`, `file-search`, `file-watcher` |

## Main CLI Dispatch

The user-facing binary is a multi-command CLI. The `MultitoolCli` parser
combines shared options, feature toggles, remote options, interactive TUI
options, and an optional subcommand. If no subcommand is provided, options are
forwarded to the interactive CLI/TUI.

Concrete command surfaces are visible in `Subcommand`:

- `Exec`
- `Review`
- `Login`
- `Logout`
- `Mcp`
- `Plugin`
- `McpServer`
- `AppServer`
- `RemoteControl`
- `App`
- `Completion`
- `Update`
- `Doctor`
- `Sandbox`
- `Debug`
- `Execpolicy`
- `Apply`
- session resume and related commands later in the same enum

Evidence:

- `codex-rs/cli/src/main.rs:91` defines Codex CLI behavior.
- `codex-rs/cli/src/main.rs:106` defines `MultitoolCli`.
- `codex-rs/cli/src/main.rs:123` defines `Subcommand`.
- `codex-rs/cli/src/main.rs:125` through `codex-rs/cli/src/main.rs:170`
  lists major user-facing subcommands.

## Why This Layout Matters

If you were building this system from scratch, you would not start by writing
one big agent file. The repository separates:

- product surfaces from core agent logic
- protocol DTOs from handlers
- local execution from remote execution
- persisted history from live session orchestration
- extension/plugin discovery from model turn execution

That separation lets the same core agent support the TUI, app-server clients,
remote environments, tests, and future integrations.
