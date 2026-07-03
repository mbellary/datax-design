# Storage and State

## Purpose

Describe canonical state, derived projections, retention, consistency, and
ownership boundaries.

## State Inventory

| Store | Data Owned | Consistency | Retention | Notes |
|---|---|---|---|---|
| Primary Database | TBD | TBD | TBD | TBD |
| Object Store | TBD | TBD | TBD | TBD |
| Cache | TBD | TBD | TBD | TBD |

## State Flow

```mermaid
flowchart LR
    Command["Command"] --> Validate["Validate"]
    Validate --> Write["Write canonical state"]
    Write --> Emit["Emit event"]
    Emit --> Project["Update projections"]
    Project --> Read["Serve reads"]
```

## Invariants

- TBD

## Backup and Recovery

- TBD
