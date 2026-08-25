# Architecture Decision Records (ADRs)

This directory contains the **Architecture Decision Records (ADRs)** for the Travel Booking Platform.

An ADR documents an important architectural decision, including:

- The problem or context.
- The decision that was made.
- Alternative approaches that were considered.
- The consequences of the decision.

These records provide historical context and explain **why** architectural decisions were made, not just **what** was implemented.

---

## ADR Index

| ADR | Title | Status |
|------|-------|--------|
| ADR-001 | Use Modular Monolith Architecture | ✅ Accepted |
| ADR-002 | Use Provider Adapter Pattern | ✅ Accepted |
| ADR-003 | Use Redis for Search Cache | ✅ Accepted |
| ADR-004 | Revalidate Offers Before Booking | ✅ Accepted |
| ADR-005 | Separate Flight and Hotel Booking Entities | ✅ Accepted |
| ADR-006 | Store Booking Snapshots | ✅ Accepted |
| ADR-007 | Use Reverse Foreign Key for Booking Transactions | ✅ Accepted |
| ADR-008 | Separate Search from Booking | ✅ Accepted |
| ADR-009 | Use Cache-Aside Strategy | ✅ Accepted |
| ADR-010 | Introduce Provider-Level Cache When Needed | 🟡 Proposed |
| ADR-011 | Redis Is Not the Source of Truth | ✅ Accepted |
| ADR-012 | Separate User Identity from Customer Profile | ✅ Accepted |

---

## ADR Status

The following status values are used throughout this directory:

| Status | Meaning |
|---------|---------|
| **Proposed** | The decision has been identified but is not yet adopted. |
| **Accepted** | The decision has been approved and is part of the current architecture. |
| **Superseded** | The decision has been replaced by a newer ADR. |
| **Deprecated** | The decision is no longer recommended but may still exist in older parts of the system. |

---

## ADR Template

Each ADR follows the same structure:

```text
Title

Status

Context

Decision

Alternatives Considered

Consequences

Related Documents
```

---

## Related Documentation

The ADRs complement the main architecture documentation found in the `/docs` directory, including:

- System Context
- Container Diagram
- Backend Component Diagram
- Business Rules
- Domain Model
- Search Architecture
- Booking Orchestration
- Caching Strategy
- Database Design
- Security
- Observability

Together, these documents describe both **how** the platform is designed and **why** key architectural decisions were made.