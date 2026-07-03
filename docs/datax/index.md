# DataX Design Overview

This section is the primary home for DataX architecture and design artifacts.
It is intentionally organized around how engineers read systems: context first,
then containers, workflows, contracts, state, operations, security, and
decisions.

## Top-Level Navigation

<div class="grid cards" markdown>

### System Context

Define the product boundary, users, external systems, and integration points.

[Open system context](system-context.md)

### Architecture

Document the container model, core components, APIs, storage, and state.

[Open container architecture](architecture/container-architecture.md)

### Flows

Capture sequence diagrams, business workflows, data movement, and failure
paths.

[Open core workflows](flows/core-workflows.md)

### Operations

Describe runtime operations, security, governance, reliability, deployment,
observability, and ownership.

[Open operational model](operations/operational-model.md)

### Decisions

Record architectural choices with context, consequences, and alternatives.

[Open ADR index](decisions/index.md)

</div>

## Design Maturity

| Area | Status | Next Step |
|---|---|---|
| System context | Ready for input | Define actors and external systems |
| Container architecture | Ready for input | Establish first container diagram |
| Core workflows | Ready for input | Capture the highest-value user journeys |
| Data flows | Ready for input | Identify producers, consumers, and stores |
| Operations | Ready for input | Define deployment and support model |
| Security | Ready for input | Capture trust boundaries and controls |
| ADRs | Ready | Start with documentation and publishing decisions |

## Starting Point

Begin with [System Context](system-context.md), then create the first
[Container Architecture](architecture/container-architecture.md). Once those
are clear, add workflows and ADRs for decisions that shape implementation.
