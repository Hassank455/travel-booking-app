# ADR-005: Separate Flight and Hotel Booking Entities

## Status

Accepted

---

## Context

Flights and hotels represent different business domains.

Although both are "bookings", they have different:

- Validation rules
- Provider workflows
- Business attributes
- Status transitions
- Snapshot requirements
- Future evolution

Attempting to combine them into a single booking table would introduce many nullable fields and increase coupling.

---

## Decision

The platform will maintain separate booking entities.

```
Flight Booking

Hotel Booking
```

Each booking type owns its own lifecycle, business rules, and related entities.

Both booking types may reference a shared Transaction entity.

---

## Alternatives Considered

### Single Booking Table

Store every booking type in one table.

**Pros**

- Fewer tables.
- Simple initial design.

**Cons**

- Many nullable columns.
- Different domains become coupled.
- Difficult future extensions.

---

### Parent / Child Tables

Use inheritance between Booking and specialized booking tables.

**Pros**

- Shared common fields.

**Cons**

- More joins.
- Higher complexity.
- Less flexibility.

---

## Consequences

### Positive

- Better separation of concerns.
- Easier maintenance.
- Better scalability.
- Clear business ownership.
- Easier future booking types.

### Negative

- Some duplicated fields.
- More tables.

---

## Related Documents

- `07-domain-model.md`
- `09-booking-orchestration.md`
- `12-database-design.md`