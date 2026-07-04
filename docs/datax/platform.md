# Platform

## Purpose

datax is an app-server which allows developers to build rich UI clients to Plan, Build, Deploy, Schedule, Execute and Monitor the Data Engineering tasks. It has a first-class integration with codex app-server.


## Actors

| Actor | Goal | Notes |
|---|---|---|
| Product user | TBD | Primary human user of DataX |
| Platform Owner | TBD | Owns System Design and steering the development work. |
| Codex | TBD | Builds and maintains the platform |
| External system | TBD | Integrates with codex app-server |

## Product Context

```mermaid
flowchart LR
    User["Product User"] --> ClientInterface["Desktop/Web/CLI"]
    ClientInterface --> DataX["datax app-server"]
    DataX --> Codex["codex app-server"]
```

## Platform Boundary

datax owns:

- Product Features required for managing the Data Engineering landscape.
    - Planning, Building, Deployment and Monitoring Data Engineering requirements.
    - Plugins support for Data Engineering Tools.
    - Worktrees to keep parallel task changes isolated with built-in Git worktree support.
    - Git worktree Support.
    - Automations for scheduling recurring tasks , or wakeup the same thread for ongoing checks.
    - Data Engineering Skills for reusing instructions and workflows in the application.
    - Data Engineering Workflows.
    - Context usage and Compaction.
    - Task Scheduler and Runner support for scheduling and executing tasks.
    - Adapter support for log collection from deployed cloud services.
    - Unified Monitoring support of the DE pipeline.
- App-Server protocol for clients.
- Firs-class datax client integration with codex app-server


datax integrates with:

- Upstream : Integrates with the client Interface.
- Downstream : Integrates with the codex app-server.

datax does not own:

- Client Interface development and deployment.

## Open Questions

- Which users and systems are in scope for the first release?
- Which integrations are mandatory versus optional?
- Which service owns each data contract?
