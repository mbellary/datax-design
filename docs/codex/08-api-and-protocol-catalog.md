# Layer 6 - API and Protocol Catalog

This layer answers: what APIs exist, what request/response methods and payload
types are involved, and how those requests are routed. It is intentionally a
catalog, not a flow narrative. Layer 7 will walk the end-to-end flows.

## Source Map

Primary source files:

- `codex-rs/app-server/README.md`: public app-server method inventory and
  behavioral notes.
- `codex-rs/app-server-protocol/src/rpc.rs`: JSON-RPC envelope structs.
- `codex-rs/app-server-protocol/src/protocol/common.rs`: typed
  `ClientRequest`, `ServerRequest`, `ServerNotification`, and
  `ClientNotification` definitions.
- `codex-rs/app-server/src/message_processor.rs`: app-server request dispatch.
- `codex-rs/app-server-transport/src/transport/mod.rs`: transport event model,
  incoming JSON parsing, overload handling, outgoing serialization.
- `codex-rs/app-server-client/README.md`: typed in-process app-server client.
- `codex-rs/protocol/src/protocol.rs`: core `Op`, `Event`, and `EventMsg`.
- `codex-rs/exec-server-protocol/src/protocol.rs`: exec-server JSON-RPC
  method constants and payload structs.
- `codex-rs/exec-server/src/server/registry.rs`: exec-server route table.
- `codex-rs/exec-server/src/rpc.rs`: generic exec-server `RpcRouter`.
- `sdk/typescript/README.md`: TypeScript SDK JSONL-over-stdio boundary.

## API Families

| Family | Transport | Primary caller | Protocol owner | Purpose |
|---|---|---|---|---|
| CLI commands | Process args/stdin/stdout | Human terminal, scripts, SDK wrapper | `codex-rs/cli` plus subcrates | Select product surface: TUI, exec, review, app-server, daemon, MCP, cloud, etc. |
| app-server external API | JSON objects over stdio, websocket, Unix socket, remote control | Desktop, IDEs, rich clients | `codex-app-server-protocol` | Thread lifecycle, turn execution, storage reads, filesystem/process utilities, plugins, MCP, account/config. |
| app-server in-process API | Tokio typed channels | `codex-tui`, `codex-exec` | `codex-app-server-client` | Same app-server semantics without a subprocess or JSON boundary. |
| core session protocol | In-process Rust enums | app-server processors, exec/TUI adapters | `codex-protocol` | Submit work to the agent and receive agent events. |
| exec-server API | JSON-RPC over local websocket or remote relay | app-server/core remote executor clients | `codex-exec-server-protocol` | Run processes, read/write files, query remote environment info, perform executor-side HTTP. |
| model provider API | HTTP/SSE/WebSocket | `codex-core` through `codex-api` | `codex-api`, `codex-client` | Send model requests and stream response events. |
| MCP protocol | JSON-RPC/RMCP | Codex MCP host/client | `codex-mcp`, `codex-mcp-server` | Discover tools/resources/prompts and execute MCP server tool calls. |
| SDK boundary | JSONL over CLI stdio | TypeScript/Python library users | SDK packages | Embed Codex by spawning `codex` and consuming structured events. |
| persistence API | Rust trait calls | app-server/thread processors/core | `codex-thread-store` | Create/resume/read/list/update/archive/delete stored threads. |

## App-Server Wire Envelope

`codex-rs/app-server-protocol/src/rpc.rs` defines the external envelope:

- `RequestId`: string or integer.
- `JSONRPCMessage`: untagged enum of `Request`, `Notification`, `Response`, and
  `Error`.
- `JSONRPCRequest`: `{ id, method, params?, trace? }`.
- `JSONRPCNotification`: `{ method, params? }`.
- `JSONRPCResponse`: `{ id, result }`.
- `JSONRPCError`: `{ id, error }`, where `error` has `code`, optional `data`,
  and `message`.

Important detail: the file says this is not true JSON-RPC 2.0 because the wire
format does not send or expect the `"jsonrpc": "2.0"` field. The constant
`JSONRPC_VERSION` exists, but the actual structs omit that field.

Transport parsing is in `codex-rs/app-server-transport/src/transport/mod.rs`:

1. `forward_incoming_message` parses a payload string into `JSONRPCMessage`.
2. `enqueue_incoming_message` wraps it in `TransportEvent::IncomingMessage`.
3. Full incoming queues return an overload error for requests instead of
   growing unbounded.
4. `serialize_outgoing_message` converts typed outgoing messages back to JSON.

Connection origins are explicit: `Stdio`, `InProcess`, `WebSocket`, and
`RemoteControl`.

## App-Server Typed Protocol

`codex-rs/app-server-protocol/src/protocol/common.rs` generates the typed
request and notification enums from macro inventories.

Client to server:

- `ClientRequest`: request methods that expect responses.
- `ClientNotification`: notification methods that do not expect responses.
- Payload naming follows `*Params` for request payloads and `*Response` for
  responses.
- v2 API payloads are under `codex-rs/app-server-protocol/src/protocol/v2/`.

Server to client:

- `ServerRequest`: server-initiated requests that require a client response.
- `ServerResponse`: typed client replies to those server requests.
- `ServerNotification`: streaming/lifecycle notifications that do not require a
  response.

Current server-to-client request methods:

| Method | Params | Response | Meaning |
|---|---|---|---|
| `item/commandExecution/requestApproval` | `CommandExecutionRequestApprovalParams` | `CommandExecutionRequestApprovalResponse` | Ask client to approve a command execution item. |
| `item/fileChange/requestApproval` | `FileChangeRequestApprovalParams` | `FileChangeRequestApprovalResponse` | Ask client to approve a file change. |
| `item/tool/requestUserInput` | `ToolRequestUserInputParams` | `ToolRequestUserInputResponse` | Ask user questions for a tool call. |
| `mcpServer/elicitation/request` | `McpServerElicitationRequestParams` | `McpServerElicitationRequestResponse` | Resolve MCP elicitation. |
| `item/permissions/requestApproval` | `PermissionsRequestApprovalParams` | `PermissionsRequestApprovalResponse` | Ask for additional permissions. |
| `item/tool/call` | `DynamicToolCallParams` | `DynamicToolCallResponse` | Ask the client to execute a dynamic tool. |
| `account/chatgptAuthTokens/refresh` | `ChatgptAuthTokensRefreshParams` | `ChatgptAuthTokensRefreshResponse` | Ask client to refresh ChatGPT auth tokens. |
| `attestation/generate` | `AttestationGenerateParams` | `AttestationGenerateResponse` | Ask client to generate attestation. |
| `currentTime/read` | `CurrentTimeReadParams` | `CurrentTimeReadResponse` | Ask client for an external clock value. |

Current client-to-server notification inventory is small: `Initialized`.

## App-Server Method Catalog

All methods below are defined by `client_request_definitions!` in
`codex-rs/app-server-protocol/src/protocol/common.rs`. The `serialization`
column matters because it tells the server how to serialize conflicting
requests: by thread id, process id, global key, filesystem watch id, or no lock.

### Thread and Memory Methods

| Method | Params | Response | Serialization | Dispatcher |
|---|---|---|---|---|
| `thread/start` | `ThreadStartParams` | `ThreadStartResponse` | none | `thread_processor.thread_start` |
| `thread/resume` | `ThreadResumeParams` | `ThreadResumeResponse` | `thread_or_path` | `thread_processor.thread_resume` |
| `thread/fork` | `ThreadForkParams` | `ThreadForkResponse` | `thread_or_path` | `thread_processor.thread_fork` |
| `thread/archive` | `ThreadArchiveParams` | `ThreadArchiveResponse` | `thread_id` | `thread_processor.thread_archive` |
| `thread/delete` | `ThreadDeleteParams` | `ThreadDeleteResponse` | `thread_id` | `thread_processor.thread_delete` |
| `thread/unsubscribe` | `ThreadUnsubscribeParams` | `ThreadUnsubscribeResponse` | `thread_id` | `thread_processor.thread_unsubscribe` |
| `thread/increment_elicitation` | `ThreadIncrementElicitationParams` | `ThreadIncrementElicitationResponse` | `thread_id` | `thread_processor.thread_increment_elicitation` |
| `thread/decrement_elicitation` | `ThreadDecrementElicitationParams` | `ThreadDecrementElicitationResponse` | `thread_id` | `thread_processor.thread_decrement_elicitation` |
| `thread/name/set` | `ThreadSetNameParams` | `ThreadSetNameResponse` | `thread_id` | `thread_processor.thread_set_name` |
| `thread/goal/set` | `ThreadGoalSetParams` | `ThreadGoalSetResponse` | `thread_id` | `thread_goal_processor.thread_goal_set` |
| `thread/goal/get` | `ThreadGoalGetParams` | `ThreadGoalGetResponse` | `thread_id` | `thread_goal_processor.thread_goal_get` |
| `thread/goal/clear` | `ThreadGoalClearParams` | `ThreadGoalClearResponse` | `thread_id` | `thread_goal_processor.thread_goal_clear` |
| `thread/metadata/update` | `ThreadMetadataUpdateParams` | `ThreadMetadataUpdateResponse` | `thread_id` | `thread_processor.thread_metadata_update` |
| `thread/settings/update` | `ThreadSettingsUpdateParams` | `ThreadSettingsUpdateResponse` | `thread_id` | `turn_processor.thread_settings_update` |
| `thread/memoryMode/set` | `ThreadMemoryModeSetParams` | `ThreadMemoryModeSetResponse` | `thread_id` | `thread_processor.thread_memory_mode_set` |
| `memory/reset` | no params | `MemoryResetResponse` | `global("memory")` | `thread_processor.memory_reset` |
| `thread/unarchive` | `ThreadUnarchiveParams` | `ThreadUnarchiveResponse` | `thread_id` | `thread_processor.thread_unarchive` |
| `thread/compact/start` | `ThreadCompactStartParams` | `ThreadCompactStartResponse` | `thread_id` | `thread_processor.thread_compact_start` |
| `thread/shellCommand` | `ThreadShellCommandParams` | `ThreadShellCommandResponse` | `thread_id` | `thread_processor.thread_shell_command` |
| `thread/approveGuardianDeniedAction` | `ThreadApproveGuardianDeniedActionParams` | `ThreadApproveGuardianDeniedActionResponse` | `thread_id` | `thread_processor.thread_approve_guardian_denied_action` |
| `thread/backgroundTerminals/clean` | `ThreadBackgroundTerminalsCleanParams` | `ThreadBackgroundTerminalsCleanResponse` | `thread_id` | `thread_processor.thread_background_terminals_clean` |
| `thread/backgroundTerminals/list` | `ThreadBackgroundTerminalsListParams` | `ThreadBackgroundTerminalsListResponse` | `thread_id` | `thread_processor.thread_background_terminals_list` |
| `thread/backgroundTerminals/terminate` | `ThreadBackgroundTerminalsTerminateParams` | `ThreadBackgroundTerminalsTerminateResponse` | `thread_id` | `thread_processor.thread_background_terminals_terminate` |
| `thread/rollback` | `ThreadRollbackParams` | `ThreadRollbackResponse` | `thread_id` | `thread_processor.thread_rollback` |
| `thread/list` | `ThreadListParams` | `ThreadListResponse` | none | `thread_processor.thread_list` |
| `thread/search` | `ThreadSearchParams` | `ThreadSearchResponse` | none | `thread_processor.thread_search` |
| `thread/loaded/list` | `ThreadLoadedListParams` | `ThreadLoadedListResponse` | none | `thread_processor.thread_loaded_list` |
| `thread/read` | `ThreadReadParams` | `ThreadReadResponse` | `thread_id` | `thread_processor.thread_read` |
| `thread/turns/list` | `ThreadTurnsListParams` | `ThreadTurnsListResponse` | none | `thread_processor.thread_turns_list` |
| `thread/items/list` | `ThreadItemsListParams` | `ThreadItemsListResponse` | none | `thread_processor.thread_items_list` |
| `thread/inject_items` | `ThreadInjectItemsParams` | `ThreadInjectItemsResponse` | `thread_id` | `turn_processor.thread_inject_items` |

Storage/update implications:

- `thread/start`, `thread/resume`, and `thread/fork` materialize a thread and
  subscribe the connection to its turn/item stream.
- `thread/list`, `thread/read`, `thread/turns/list`, and `thread/items/list`
  read persisted thread state.
- `thread/metadata/update`, `thread/name/set`, `thread/goal/*`,
  `thread/memoryMode/set`, archive/unarchive/delete, and `memory/reset` mutate
  persisted metadata or files.
- `thread/settings/update`, `thread/compact/start`, `thread/shellCommand`,
  background-terminal methods, rollback, and inject-items affect a loaded
  session and may later persist rollout items or metadata.

### Turn, Review, and Realtime Methods

| Method | Params | Response | Serialization | Dispatcher |
|---|---|---|---|---|
| `turn/start` | `TurnStartParams` | `TurnStartResponse` | `thread_id` | `turn_processor.turn_start` |
| `turn/steer` | `TurnSteerParams` | `TurnSteerResponse` | `thread_id` | `turn_processor.turn_steer` |
| `turn/interrupt` | `TurnInterruptParams` | `TurnInterruptResponse` | `thread_id` | `turn_processor.turn_interrupt` |
| `thread/realtime/start` | `ThreadRealtimeStartParams` | `ThreadRealtimeStartResponse` | `thread_id` | `turn_processor.thread_realtime_start` |
| `thread/realtime/appendAudio` | `ThreadRealtimeAppendAudioParams` | `ThreadRealtimeAppendAudioResponse` | `thread_id` | `turn_processor.thread_realtime_append_audio` |
| `thread/realtime/appendText` | `ThreadRealtimeAppendTextParams` | `ThreadRealtimeAppendTextResponse` | `thread_id` | `turn_processor.thread_realtime_append_text` |
| `thread/realtime/appendSpeech` | `ThreadRealtimeAppendSpeechParams` | `ThreadRealtimeAppendSpeechResponse` | `thread_id` | `turn_processor.thread_realtime_append_speech` |
| `thread/realtime/stop` | `ThreadRealtimeStopParams` | `ThreadRealtimeStopResponse` | `thread_id` | `turn_processor.thread_realtime_stop` |
| `thread/realtime/listVoices` | `ThreadRealtimeListVoicesParams` | `ThreadRealtimeListVoicesResponse` | none | `turn_processor.thread_realtime_list_voices` |
| `review/start` | `ReviewStartParams` | `ReviewStartResponse` | `thread_id` | `turn_processor.review_start` |

Request payloads in this family become core `Op` submissions. For example,
`turn/start` maps to `Op::UserInput`; `turn/interrupt` maps to `Op::Interrupt`;
realtime methods map to `Op::RealtimeConversation*`.

### Catalog, Plugin, Skill, Hook, and App Methods

| Method | Params | Response | Serialization | Dispatcher |
|---|---|---|---|---|
| `skills/list` | `SkillsListParams` | `SkillsListResponse` | `global_shared_read("config")` | `catalog_processor.skills_list` |
| `skills/extraRoots/set` | `SkillsExtraRootsSetParams` | `SkillsExtraRootsSetResponse` | `global("config")` | `catalog_processor.skills_extra_roots_set` |
| `hooks/list` | `HooksListParams` | `HooksListResponse` | `global("config")` | `catalog_processor.hooks_list` |
| `marketplace/add` | `MarketplaceAddParams` | `MarketplaceAddResponse` | `global("config")` | `marketplace_processor.marketplace_add` |
| `marketplace/remove` | `MarketplaceRemoveParams` | `MarketplaceRemoveResponse` | `global("config")` | `marketplace_processor.marketplace_remove` |
| `marketplace/upgrade` | `MarketplaceUpgradeParams` | `MarketplaceUpgradeResponse` | `global("config")` | `marketplace_processor.marketplace_upgrade` |
| `plugin/list` | `PluginListParams` | `PluginListResponse` | none | `plugin_processor.plugin_list` |
| `plugin/installed` | `PluginInstalledParams` | `PluginInstalledResponse` | none | `plugin_processor.plugin_installed` |
| `plugin/read` | `PluginReadParams` | `PluginReadResponse` | none | `plugin_processor.plugin_read` |
| `plugin/skill/read` | `PluginSkillReadParams` | `PluginSkillReadResponse` | `global("config")` | `plugin_processor.plugin_skill_read` |
| `plugin/share/save` | `PluginShareSaveParams` | `PluginShareSaveResponse` | `global("config")` | `plugin_processor.plugin_share_save` |
| `plugin/share/updateTargets` | `PluginShareUpdateTargetsParams` | `PluginShareUpdateTargetsResponse` | `global("config")` | `plugin_processor.plugin_share_update_targets` |
| `plugin/share/list` | `PluginShareListParams` | `PluginShareListResponse` | `global("config")` | `plugin_processor.plugin_share_list` |
| `plugin/share/checkout` | `PluginShareCheckoutParams` | `PluginShareCheckoutResponse` | `global("config")` | `plugin_processor.plugin_share_checkout` |
| `plugin/share/delete` | `PluginShareDeleteParams` | `PluginShareDeleteResponse` | `global("config")` | `plugin_processor.plugin_share_delete` |
| `app/list` | `AppsListParams` | `AppsListResponse` | none | `apps_processor.apps_list` |
| `skills/config/write` | `SkillsConfigWriteParams` | `SkillsConfigWriteResponse` | `global("config")` | `catalog_processor.skills_config_write` |
| `plugin/install` | `PluginInstallParams` | `PluginInstallResponse` | `global("config")` | `plugin_processor.plugin_install` |
| `plugin/uninstall` | `PluginUninstallParams` | `PluginUninstallResponse` | `global("config")` | `plugin_processor.plugin_uninstall` |

These methods primarily read or mutate local plugin/skill/app catalogs and user
configuration. They surface in rich clients for plugin pages, mention pickers,
skill selectors, hook views, and app connector pickers.

### Filesystem Methods

| Method | Params | Response | Serialization | Dispatcher |
|---|---|---|---|---|
| `fs/readFile` | `FsReadFileParams` | `FsReadFileResponse` | none | `fs_processor.read_file` |
| `fs/writeFile` | `FsWriteFileParams` | `FsWriteFileResponse` | none | `fs_processor.write_file` |
| `fs/createDirectory` | `FsCreateDirectoryParams` | `FsCreateDirectoryResponse` | none | `fs_processor.create_directory` |
| `fs/getMetadata` | `FsGetMetadataParams` | `FsGetMetadataResponse` | none | `fs_processor.get_metadata` |
| `fs/readDirectory` | `FsReadDirectoryParams` | `FsReadDirectoryResponse` | none | `fs_processor.read_directory` |
| `fs/remove` | `FsRemoveParams` | `FsRemoveResponse` | none | `fs_processor.remove` |
| `fs/copy` | `FsCopyParams` | `FsCopyResponse` | none | `fs_processor.copy` |
| `fs/watch` | `FsWatchParams` | `FsWatchResponse` | `fs_watch_id` | `fs_processor.watch` |
| `fs/unwatch` | `FsUnwatchParams` | `FsUnwatchResponse` | `fs_watch_id` | `fs_processor.unwatch` |

Payload type detail:

- App-server filesystem APIs use absolute host paths and base64 for file
  contents: `fs/readFile` returns `dataBase64`; `fs/writeFile` accepts
  `dataBase64`.
- `fs/watch` emits `fs/changed` notifications with `watchId` and changed paths.
- These APIs are intentionally concurrent in the protocol definition.

### Command and Process Methods

| Method | Params | Response | Serialization | Dispatcher |
|---|---|---|---|---|
| `command/exec` | `CommandExecParams` | `CommandExecResponse` | optional command process id | `command_exec_processor.one_off_command_exec` |
| `command/exec/write` | `CommandExecWriteParams` | `CommandExecWriteResponse` | command process id | `command_exec_processor.command_exec_write` |
| `command/exec/terminate` | `CommandExecTerminateParams` | `CommandExecTerminateResponse` | command process id | `command_exec_processor.command_exec_terminate` |
| `command/exec/resize` | `CommandExecResizeParams` | `CommandExecResizeResponse` | command process id | `command_exec_processor.command_exec_resize` |
| `process/spawn` | `ProcessSpawnParams` | `ProcessSpawnResponse` | process handle | `process_exec_processor.process_spawn` |
| `process/writeStdin` | `ProcessWriteStdinParams` | `ProcessWriteStdinResponse` | process handle | `process_exec_processor.process_write_stdin` |
| `process/kill` | `ProcessKillParams` | `ProcessKillResponse` | process handle | `process_exec_processor.process_kill` |
| `process/resizePty` | `ProcessResizePtyParams` | `ProcessResizePtyResponse` | process handle | `process_exec_processor.process_resize_pty` |

`command/exec` is sandboxed as a utility API. `process/spawn` is an
experimental standalone host process API. Both stream base64 output deltas via
notifications.

### Model, Feature, Permission, Environment, and Collaboration Methods

| Method | Params | Response | Serialization | Dispatcher |
|---|---|---|---|---|
| `model/list` | `ModelListParams` | `ModelListResponse` | none | `catalog_processor.model_list` |
| `modelProvider/capabilities/read` | `ModelProviderCapabilitiesReadParams` | `ModelProviderCapabilitiesReadResponse` | none | `config_processor.model_provider_capabilities_read` |
| `experimentalFeature/list` | `ExperimentalFeatureListParams` | `ExperimentalFeatureListResponse` | `global("config")` | `catalog_processor.experimental_feature_list` |
| `permissionProfile/list` | `PermissionProfileListParams` | `PermissionProfileListResponse` | `global_shared_read("config")` | `catalog_processor.permission_profile_list` |
| `experimentalFeature/enablement/set` | `ExperimentalFeatureEnablementSetParams` | `ExperimentalFeatureEnablementSetResponse` | `global("config")` | `config_processor.experimental_feature_enablement_set` |
| `collaborationMode/list` | `CollaborationModeListParams` | `CollaborationModeListResponse` | none | `catalog_processor.collaboration_mode_list` |
| `environment/add` | `EnvironmentAddParams` | `EnvironmentAddResponse` | `global("environment")` | `environment_processor.environment_add` |
| `environment/info` | `EnvironmentInfoParams` | `EnvironmentInfoResponse` | `global_shared_read("environment")` | `environment_processor.environment_info` |

These methods let clients populate pickers and validate runtime capability:
models, provider capabilities, feature flags, permission profiles,
collaboration modes, and remote execution environments.

### MCP Methods

| Method | Params | Response | Serialization | Dispatcher |
|---|---|---|---|---|
| `mcpServer/oauth/login` | `McpServerOauthLoginParams` | `McpServerOauthLoginResponse` | MCP OAuth server key | `mcp_processor.mcp_server_oauth_login` |
| `config/mcpServer/reload` | no params | `McpServerRefreshResponse` | `global("mcp-registry")` | `mcp_processor.mcp_server_refresh` |
| `mcpServerStatus/list` | `ListMcpServerStatusParams` | `ListMcpServerStatusResponse` | `global("mcp-registry")` | `mcp_processor.mcp_server_status_list` |
| `mcpServer/resource/read` | `McpResourceReadParams` | `McpResourceReadResponse` | optional thread id | `mcp_processor.mcp_resource_read` |
| `mcpServer/tool/call` | `McpServerToolCallParams` | `McpServerToolCallResponse` | `thread_id` | `mcp_processor.mcp_server_tool_call` |

These bridge rich clients to MCP status/resource/tool operations. Core MCP tool
mutation is not implemented ad hoc in the API layer; the MCP connection manager
and catalog crates own server connection and tool catalog state.

### Remote Control and Account Methods

| Method | Params | Response | Serialization | Dispatcher |
|---|---|---|---|---|
| `remoteControl/enable` | nullable `RemoteControlEnableParams` | `RemoteControlEnableResponse` | `global("remote-control")` | `remote_control_processor.enable` |
| `remoteControl/disable` | nullable `RemoteControlDisableParams` | `RemoteControlDisableResponse` | `global("remote-control")` | `remote_control_processor.disable` |
| `remoteControl/status/read` | no params | `RemoteControlStatusReadResponse` | `global_shared_read("remote-control")` | `remote_control_processor.status_read` |
| `remoteControl/pairing/start` | `RemoteControlPairingStartParams` | `RemoteControlPairingStartResponse` | `global("remote-control-pairing")` | `remote_control_processor.pairing_start` |
| `remoteControl/pairing/status` | `RemoteControlPairingStatusParams` | `RemoteControlPairingStatusResponse` | `global_shared_read("remote-control-pairing")` | `remote_control_processor.pairing_status` |
| `remoteControl/client/list` | `RemoteControlClientsListParams` | `RemoteControlClientsListResponse` | `global_shared_read("remote-control-clients")` | `remote_control_processor.clients_list` |
| `remoteControl/client/revoke` | `RemoteControlClientsRevokeParams` | `RemoteControlClientsRevokeResponse` | `global("remote-control-clients")` | `remote_control_processor.clients_revoke` |
| `account/login/start` | `LoginAccountParams` | `LoginAccountResponse` | `global("account-auth")` | `account_processor.login_account` |
| `account/login/cancel` | `CancelLoginAccountParams` | `CancelLoginAccountResponse` | `global("account-auth")` | `account_processor.cancel_login_account` |
| `account/logout` | no params | `LogoutAccountResponse` | `global("account-auth")` | `account_processor.logout_account` |
| `account/read` | `GetAccountParams` | `GetAccountResponse` | `global("account-auth")` | `account_processor.get_account` |
| `account/rateLimits/read` | no params | `GetAccountRateLimitsResponse` | none | `account_processor.get_account_rate_limits` |
| `account/rateLimitResetCredit/consume` | `ConsumeAccountRateLimitResetCreditParams` | `ConsumeAccountRateLimitResetCreditResponse` | `global("account-auth")` | `account_processor.consume_account_rate_limit_reset_credit` |
| `account/usage/read` | no params | `GetAccountTokenUsageResponse` | none | `account_processor.get_account_token_usage` |
| `account/workspaceMessages/read` | no params | `GetWorkspaceMessagesResponse` | none | `account_processor.get_workspace_messages` |
| `account/sendAddCreditsNudgeEmail` | `SendAddCreditsNudgeEmailParams` | `SendAddCreditsNudgeEmailResponse` | `global("account-auth")` | `account_processor.send_add_credits_nudge_email` |

### Config, Migration, Windows, Feedback, and Search Methods

| Method | Params | Response | Serialization | Dispatcher |
|---|---|---|---|---|
| `config/read` | `ConfigReadParams` | `ConfigReadResponse` | `global_shared_read("config")` | `config_processor.read` |
| `config/value/write` | `ConfigValueWriteParams` | `ConfigWriteResponse` | `global("config")` | `config_processor.value_write` |
| `config/batchWrite` | `ConfigBatchWriteParams` | `ConfigWriteResponse` | `global("config")` | `config_processor.batch_write` |
| `configRequirements/read` | no params | `ConfigRequirementsReadResponse` | `global("config")` | `config_processor.config_requirements_read` |
| `externalAgentConfig/detect` | `ExternalAgentConfigDetectParams` | `ExternalAgentConfigDetectResponse` | `global("config")` | `external_agent_config_processor.detect` |
| `externalAgentConfig/import` | `ExternalAgentConfigImportParams` | `ExternalAgentConfigImportResponse` | `global("config")` | `external_agent_config_processor.import` |
| `externalAgentConfig/import/readHistories` | no params | `ExternalAgentConfigImportHistoriesReadResponse` | `global_shared_read("config")` | `external_agent_config_processor.read_import_histories` |
| `windowsSandbox/setupStart` | `WindowsSandboxSetupStartParams` | `WindowsSandboxSetupStartResponse` | `global("windows-sandbox-setup")` | `windows_sandbox_processor.windows_sandbox_setup_start` |
| `windowsSandbox/readiness` | no params | `WindowsSandboxReadinessResponse` | `global("config")` | `windows_sandbox_processor.windows_sandbox_readiness` |
| `feedback/upload` | `FeedbackUploadParams` | `FeedbackUploadResponse` | none | `feedback_processor.feedback_upload` |
| legacy `FuzzyFileSearch` | `FuzzyFileSearchParams` | `FuzzyFileSearchResponse` | none | `search_processor.fuzzy_file_search` |
| `fuzzyFileSearch/sessionStart` | `FuzzyFileSearchSessionStartParams` | `FuzzyFileSearchSessionStartResponse` | fuzzy session id | `search_processor.fuzzy_file_search_session_start_response` |
| `fuzzyFileSearch/sessionUpdate` | `FuzzyFileSearchSessionUpdateParams` | `FuzzyFileSearchSessionUpdateResponse` | fuzzy session id | `search_processor.fuzzy_file_search_session_update_response` |
| `fuzzyFileSearch/sessionStop` | `FuzzyFileSearchSessionStopParams` | `FuzzyFileSearchSessionStopResponse` | fuzzy session id | `search_processor.fuzzy_file_search_session_stop` |

## App-Server Notifications

`ServerNotification` is the app-server's streaming surface. Important groups:

- Thread lifecycle: `thread/started`, `thread/status/changed`,
  `thread/archived`, `thread/deleted`, `thread/unarchived`, `thread/closed`,
  `thread/name/updated`, `thread/goal/updated`, `thread/goal/cleared`,
  `thread/settings/updated`, `thread/tokenUsage/updated`.
- Turn lifecycle: `turn/started`, `turn/completed`, `turn/diff/updated`,
  `turn/plan/updated`, `thread/compacted`.
- Item lifecycle: `item/started`, `item/completed`,
  `rawResponseItem/completed`, `item/agentMessage/delta`,
  `item/plan/delta`, reasoning deltas, command execution deltas, file change
  deltas, MCP tool progress.
- Process and command utilities: `command/exec/outputDelta`,
  `process/outputDelta`, `process/exited`.
- MCP/account/app/remote/config: OAuth completion, MCP status updates,
  account updates, app list updates, remote-control status changes, external
  import progress/completion, config warnings.
- Filesystem watchers: `fs/changed`.
- Realtime: `thread/realtime/started`, `thread/realtime/itemAdded`,
  transcript/audio deltas, SDP, error, and closed notifications.
- Windows sandbox: `windowsSandbox/setupCompleted` and world-writable warning.

These notifications are where UI surfaces receive data to render progress.
Immediate request responses usually acknowledge acceptance or return a starting
object; incremental UI state arrives through notifications.

## App-Server Routing Mechanics

The request path is:

1. A transport produces `TransportEvent::IncomingMessage`.
2. The app-server processor loop receives the event.
3. For requests, `MessageProcessor::process_request` deserializes the
   `JSONRPCRequest` into typed `ClientRequest`.
4. `handle_initialized_client_request` matches the `ClientRequest` variant and
   calls a processor method.
5. Processor methods either return a `ClientResponsePayload`, return no
   immediate payload and later emit notifications, or send errors.
6. `outgoing.send_response_as`, `outgoing.send_error`, or notification routing
   writes back through the connection writer.

The dispatch table is a large `match` in
`codex-rs/app-server/src/message_processor.rs`. Processor ownership is clear:

- `thread_processor`: thread lifecycle, stored-thread reads, archive/delete,
  shell command, rollback, memory mode.
- `thread_goal_processor`: goal set/get/clear.
- `turn_processor`: turn start/steer/interrupt, review, realtime, inject items,
  settings update.
- `fs_processor`: filesystem requests and watches.
- `command_exec_processor`: sandboxed one-off commands.
- `process_exec_processor`: standalone process API.
- `catalog_processor`: models, skills, hooks, feature flags, permissions,
  collaboration modes.
- `plugin_processor` and `marketplace_processor`: plugin and marketplace
  catalog/install/share operations.
- `mcp_processor`: MCP status/resource/tool/OAuth operations.
- `config_processor`: config read/write, requirements, provider capabilities,
  feature enablement.
- `account_processor`: auth/account/usage/rate-limit methods.
- `remote_control_processor`: remote control enable/pairing/client grant
  operations.
- `environment_processor`: remote execution environment registration/info.
- `external_agent_config_processor`, `windows_sandbox_processor`,
  `feedback_processor`, `search_processor`, `git_processor`: specialized
  support APIs.

## In-Process App-Server API

`codex-app-server-client` preserves app-server semantics without JSON at the
hot path:

- client to server: `ClientRequest` and `ClientNotification`.
- server to client: `InProcessServerEvent`, with `ServerRequest`,
  `ServerNotification`, and `LegacyNotification`.
- startup still follows normal app-server flow: initialize, `thread/start` or
  `thread/resume`, then later session metadata such as `SessionConfigured`.
- bounded queues are part of the contract; saturation yields overload or lagged
  events instead of unbounded memory growth.

This is how `codex-tui` and `codex-exec` can use app-server behavior without
requiring a separate `codex app-server` subprocess.

## Core Session Protocol

`codex-rs/protocol/src/protocol.rs` defines the in-process protocol between
frontends/app-server processors and the agent engine.

Request side: `Op`.

Important `Op` variants:

- `Interrupt`
- `CleanBackgroundTerminals`
- `RealtimeConversationStart`, `RealtimeConversationAudio`,
  `RealtimeConversationText`, `RealtimeConversationSpeech`,
  `RealtimeConversationClose`, `RealtimeConversationListVoices`
- `UserInput { items, final_output_json_schema,
  responsesapi_client_metadata, additional_context, thread_settings }`
- `ThreadSettings { thread_settings }`
- `InterAgentCommunication`
- `ExecApproval`
- `PatchApproval`
- `ResolveElicitation`
- `UserInputAnswer`
- `RequestPermissionsResponse`
- dynamic tool response variants later in the enum

Response side: `Event`.

- `Event` has `id` and `msg`.
- `id` correlates the event to a submission.
- `msg` is `EventMsg`.

Important `EventMsg` groups:

- error/warning: `Error`, `Warning`, `GuardianWarning`.
- realtime lifecycle and payload: `RealtimeConversationStarted`,
  `RealtimeConversationRealtime`, `RealtimeConversationClosed`,
  `RealtimeConversationSdp`.
- model/backend metadata: `ModelReroute`, `ModelVerification`,
  `TurnModerationMetadata`, `SafetyBuffering`.
- context/session state: `ContextCompacted`, `ThreadRolledBack`,
  `TurnStarted`, `ThreadSettingsApplied`, `TurnComplete`, `TokenCount`,
  `SessionConfigured`, `ThreadGoalUpdated`.
- user-visible content: `AgentMessage`, `UserMessage`, `AgentReasoning`,
  reasoning raw content and section breaks.

Design implication: app-server API methods are not the agent engine API. They
are adapters that translate external JSON-RPC requests into core `Op` values,
then translate `EventMsg` values back into app-server `ServerNotification`
payloads and persistent `ThreadItem` projections.

## Exec-Server API

`codex-rs/exec-server-protocol/src/protocol.rs` defines method constants:

| Method | Params | Response/notification | Meaning |
|---|---|---|---|
| `initialize` | `InitializeParams` | `InitializeResponse` | Start exec-server protocol session. |
| `initialized` | notification | none | Client marks initialization complete. |
| `process/start` | `ExecParams` | `ExecResponse` | Start process with argv, cwd URI, env policy, TTY/stdin, sandbox, managed network. |
| `process/read` | `ReadParams` | `ReadResponse` | Poll buffered process output and exit state. |
| `process/write` | `WriteParams` | `WriteResponse` | Write base64 stdin bytes. |
| `process/signal` | `SignalParams` | `SignalResponse` | Send process signal such as interrupt. |
| `process/terminate` | `TerminateParams` | `TerminateResponse` | Terminate process and report whether it was running. |
| `process/output` | `ExecOutputDeltaNotification` | notification | Stream output chunk. |
| `process/exited` | `ExecExitedNotification` | notification | Process exited. |
| `process/closed` | `ExecClosedNotification` | notification | Process output stream closed. |
| `environment/info` | unit | `EnvironmentInfo` | Return shell and default cwd URI. |
| `fs/readFile` | `FsReadFileParams` | `FsReadFileResponse` | Read file bytes as base64. |
| `fs/open` | `FsOpenParams` | `FsOpenResponse` | Open file handle. |
| `fs/readBlock` | `FsReadBlockParams` | `FsReadBlockResponse` | Read byte range from handle. |
| `fs/close` | `FsCloseParams` | `FsCloseResponse` | Close file handle. |
| `fs/writeFile` | `FsWriteFileParams` | `FsWriteFileResponse` | Write base64 bytes. |
| `fs/createDirectory` | `FsCreateDirectoryParams` | `FsCreateDirectoryResponse` | Create directory. |
| `fs/getMetadata` | `FsGetMetadataParams` | `FsGetMetadataResponse` | Return type, size, timestamps. |
| `fs/canonicalize` | `FsCanonicalizeParams` | `FsCanonicalizeResponse` | Canonicalize path URI. |
| `fs/readDirectory` | `FsReadDirectoryParams` | `FsReadDirectoryResponse` | List child entries. |
| `fs/walk` | `FsWalkParams` | `FsWalkResponse` | Recursive walk. |
| `fs/remove` | `FsRemoveParams` | `FsRemoveResponse` | Remove file/directory. |
| `fs/copy` | `FsCopyParams` | `FsCopyResponse` | Copy file/directory. |
| `http/request` | `HttpRequestParams` | `HttpRequestResponse` | Executor-side HTTP request. |
| `http/request/bodyDelta` | `HttpRequestBodyDeltaNotification` | notification | Stream HTTP response body bytes. |

Payload details:

- `ByteChunk` serializes bytes as base64.
- Paths use `PathUri` so the client can talk to executors on a different OS.
- `ExecParams.process_id` is a client-chosen logical handle, not an OS pid.
- `ReadResponse` contains `chunks`, `next_seq`, `exited`, `exit_code`,
  `closed`, `failure`, and `sandbox_denied`.
- `HttpRequestParams` can buffer a response or stream it through
  `http/request/bodyDelta`.

Routing:

- `codex-rs/exec-server/src/server/registry.rs` builds a `RpcRouter` and maps
  each method constant to an `ExecServerHandler` method.
- `codex-rs/exec-server/src/rpc.rs` implements `RpcRouter::request`,
  `request_with_id`, and `notification`. It deserializes params to typed
  payloads, invokes the handler, serializes the result, and returns either a
  response or JSON-RPC error.
- `http/request` uses `request_with_id` because streaming body notifications
  need the original JSON-RPC request id.

## SDK Boundary

The TypeScript SDK does not call app-server JSON-RPC directly. Its README says
it wraps the `codex` CLI from `@openai/codex`, spawns the CLI, and exchanges
JSONL events over stdin/stdout. User-facing SDK calls such as `startThread`,
`resumeThread`, `run`, and `runStreamed` are library abstractions over that CLI
JSONL protocol.

For a from-scratch implementation, treat the SDK as a separate adapter:

1. Define a stable CLI JSONL event stream.
2. Spawn the CLI from the SDK.
3. Convert library methods into CLI input messages.
4. Convert CLI output events into SDK `Thread`, `Turn`, and streamed event
   objects.

## Storage API Boundary

The persistence layer is not JSON-RPC. It is a Rust trait boundary. The API
layer calls into thread processors, and processors call `ThreadStore` methods
such as:

- `create_thread`
- `resume_thread`
- `append_items`
- `persist_thread`
- `flush_thread`
- `load_history`
- `read_thread`
- `list_threads`
- `list_turns`
- `list_items`
- `update_thread_metadata`
- `archive_thread`
- `unarchive_thread`
- `delete_thread`

From an API design perspective, this means external clients never write rollout
JSONL or SQLite directly. They ask the app-server for operations like
`turn/start`, `thread/metadata/update`, or `thread/read`; app-server owns the
translation into storage calls.

## Request and Payload Type Rules

The concrete rules visible in this codebase:

- External app-server messages are JSON objects using method strings such as
  `thread/start`, `turn/start`, and `fs/readFile`.
- Request payload structs are named `*Params`.
- Response structs are named `*Response`.
- Notification structs are named `*Notification`.
- Most app-server v2 API wire fields are camelCase. Config APIs are an
  intentional exception because they mirror `config.toml` keys.
- Byte payloads are base64 strings or base64-backed wrapper structs.
- Long-running work usually responds immediately and streams progress through
  notifications.
- Potentially conflicting mutations are serialized by protocol metadata:
  per-thread, per-process, per-watch, or named global locks.
- Read-only or explicitly concurrent methods use `serialization: None` or a
  shared-read key.
- Core engine requests are Rust enums, not JSON method strings.
- Exec-server paths use portable `file:` URIs, not raw local paths, because the
  executor may run on a different OS.

## How to Implement This Layer From Scratch

A minimal implementation sequence:

1. Define one envelope type:
   `Request { id, method, params }`, `Notification { method, params }`,
   `Response { id, result }`, `Error { id, error }`.
2. Define typed request/response/notification structs for each public method.
3. Generate or hand-write a `ClientRequest` enum mapping method strings to
   payload types.
4. Build a transport adapter that converts bytes or websocket text into
   envelope structs.
5. Build a dispatcher that deserializes the envelope into `ClientRequest` and
   calls a handler table.
6. Separate handlers by ownership: thread, turn, filesystem, command, plugin,
   MCP, account, config, remote-control, environment.
7. Add a notification bus so long-running work can stream state changes after
   the immediate response.
8. Add server-to-client request support for approvals, elicitation, dynamic
   tool calls, time/attestation/auth refresh.
9. Define the internal core protocol separately from the external API:
   `Op` for submissions and `Event`/`EventMsg` for outputs.
10. Define a storage trait and make API handlers call storage through that
    trait, not by writing files directly.
11. Add exec-server as a separate protocol if local and remote execution hosts
    may differ. Use URI paths and base64 bytes from the start.
12. Add schema/TypeScript generation only after the method and payload
    inventory is stable.

The non-negotiable design point is separation of contracts: external JSON-RPC
contract, internal core session contract, executor contract, and storage
contract are different APIs with different payload models.
