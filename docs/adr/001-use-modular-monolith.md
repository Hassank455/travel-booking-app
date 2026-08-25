# ADR-001: Use Modular Monolith Architecture

## Status

Accepted

---

## Context

The Travel Booking Platform is expected to support multiple business domains, including:

- Flight Search
- Hotel Search
- Flight Booking
- Hotel Booking
- Payments
- Customer Management
- Notifications

Although these domains are logically independent, they belong to the same business application and currently do not require independent deployment.

Choosing a microservices architecture at this stage would introduce unnecessary operational complexity, including:

- Distributed transactions
- Service discovery
- Inter-service communication
- Network latency
- Deployment complexity
- Operational overhead

The expected project size and team do not currently justify this complexity.

---

## Decision

The platform will adopt a **Modular Monolith** architecture.

Each business capability will be implemented as an independent module with clear boundaries while remaining part of a single deployable application.

Conceptually:

```text
Backend Application

├── Search Module
├── Booking Module
├── Payment Module
├── Customer Module
├── Notification Module
└── Shared Infrastructure
```

Modules communicate through well-defined interfaces rather than direct implementation dependencies.

---

## Alternatives Considered

### Microservices

Each domain could be deployed independently.

**Pros**

- Independent deployment.
- Independent scaling.
- Strong service isolation.

**Cons**

- High operational complexity.
- Distributed transactions.
- Network communication overhead.
- More infrastructure.
- More difficult local development.

---

### Layered Monolith

All business logic could be organized only by technical layers.

**Pros**

- Very simple.
- Easy to start.

**Cons**

- Weak business boundaries.
- Tight coupling between features.
- Difficult long-term maintenance.

---

## Consequences

### Positive

- Simpler deployment.
- Easier local development.
- Strong business boundaries.
- Easier refactoring.
- Future migration to microservices remains possible.

### Negative

- Entire application is deployed together.
- Requires discipline to maintain module boundaries.

---

## Related Documents

- `02-container-diagram.md`
- `03-backend-component-diagram.md`
- `07-domain-model.md`