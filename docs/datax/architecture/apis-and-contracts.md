# APIs and Contracts

## Purpose

Catalog public APIs, internal service contracts, event schemas, and data
exchange formats.

## Contract Inventory

| Contract | Type | Producer | Consumer | Versioning |
|---|---|---|---|---|
| TBD | REST / event / file / RPC | TBD | TBD | TBD |

## API Shape

```mermaid
sequenceDiagram
    participant Client
    participant API as DataX API
    participant Domain as Domain Service
    participant Store as Data Store

    Client->>API: Request
    API->>Domain: Validate and execute command
    Domain->>Store: Read/write state
    Store-->>Domain: Result
    Domain-->>API: Response model
    API-->>Client: Response
```

## Compatibility Rules

- TBD

## Error Model

- TBD
