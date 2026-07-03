# Template: Sequence Diagram

## Scenario

Describe the user or system scenario.

## Participants

| Participant | Role |
|---|---|
| TBD | TBD |

## Sequence

```mermaid
sequenceDiagram
    autonumber
    participant A as Actor
    participant S as Service
    participant D as Data Store

    A->>S: Request
    S->>D: Read/write
    D-->>S: Result
    S-->>A: Response
```

## Notes

- TBD

## Failure Paths

- TBD
