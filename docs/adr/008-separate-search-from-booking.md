# ADR-008: Separate Search from Booking

## Status

Accepted

---

## Context

Searching for travel offers and creating bookings are fundamentally different business capabilities.

Search is primarily a read-oriented operation that aggregates data from multiple external providers. It is optimized for performance, caching, filtering, and ranking.

Booking is a transactional operation that creates persistent business data, coordinates payments, revalidates provider offers, and guarantees a consistent business outcome.

Although these capabilities are closely related from the customer's perspective, they have different responsibilities, failure modes, and consistency requirements.

Keeping both concerns inside a single module would increase coupling and make future maintenance more difficult.

---

## Decision

The platform separates Search and Booking into two independent business modules.

The Search Module is responsible for:

- Searching external providers.
- Aggregating results.
- Normalizing provider responses.
- Applying filters and sorting.
- Managing temporary search cache.

The Booking Module is responsible for:

- Revalidating offers.
- Creating bookings.
- Coordinating transactions.
- Confirming bookings with providers.
- Persisting booking state.

Conceptually:

```text
Customer
      ↓
Search Module
      ↓
Offer Returned
      ↓
Customer Selects Offer
      ↓
Booking Module
      ↓
Offer Revalidation
      ↓
Booking Confirmation
```

The Booking Module never trusts cached search results without provider revalidation.

---

## Alternatives Considered

### Single Travel Module

Implement Search and Booking inside one large module.

**Pros**

- Simpler initial implementation.
- Fewer module boundaries.

**Cons**

- Mixed responsibilities.
- Search logic and booking logic become tightly coupled.
- Harder testing.
- Harder long-term maintenance.

---

### Booking Directly from Search

Allow the Search Module to create bookings directly.

**Pros**

- Fewer application layers.
- Lower implementation effort.

**Cons**

- Search becomes responsible for transactional operations.
- Violates separation of concerns.
- Makes future evolution more difficult.

---

## Consequences

### Positive

- Clear business boundaries.
- Independent evolution of Search and Booking.
- Easier testing.
- Better maintainability.
- Simpler observability and troubleshooting.

### Negative

- Additional coordination between modules.
- Slightly more application complexity.

---

## Related Documents

- `07-domain-model.md`
- `08-search-architecture.md`
- `09-booking-orchestration.md`
- `10-caching-strategy.md`