# Template: Container Diagram

## Scope

What runtime boundary does this diagram explain?

```mermaid
flowchart TB
    Client["Client"]
    API["API Service"]
    Worker["Worker"]
    DB[("Database")]

    Client --> API
    API --> DB
    API --> Worker
    Worker --> DB
```

## Containers

| Container | Responsibility | Runtime | Data Owned |
|---|---|---|---|
| TBD | TBD | TBD | TBD |

## Dependencies

- TBD

## Operational Notes

- TBD
