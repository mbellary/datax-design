# Core Workflows

## Purpose

Capture the most important product and system workflows as sequence diagrams.

## Workflow Inventory

| Workflow | Trigger | Outcome | Status |
|---|---|---|---|
| TBD | TBD | TBD | Draft |

## Example Sequence

```mermaid
sequenceDiagram
    autonumber
    participant User
    participant UI as DataX UI
    participant API as DataX API
    participant Worker

    User->>UI: Start workflow
    UI->>API: Submit command
    API-->>UI: Accepted
    API->>Worker: Dispatch work
    Worker-->>API: Complete work
    API-->>UI: Publish status update
```

## Failure Paths

- TBD
