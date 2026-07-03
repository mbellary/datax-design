# Security and Governance

## Purpose

Capture trust boundaries, authentication, authorization, data protection,
compliance needs, and governance controls.

## Trust Boundaries

```mermaid
flowchart TB
    User["User Boundary"] --> App["Application Boundary"]
    App --> Data["Data Boundary"]
    App --> External["External Systems Boundary"]
```

## Controls

| Area | Control | Status | Notes |
|---|---|---|---|
| Authentication | TBD | TBD | TBD |
| Authorization | TBD | TBD | TBD |
| Secrets | TBD | TBD | TBD |
| Audit | TBD | TBD | TBD |
| Data retention | TBD | TBD | TBD |

## Open Questions

- What data classification model applies to DataX?
- Which actions require audit trails?
- Which integrations cross a security boundary?
