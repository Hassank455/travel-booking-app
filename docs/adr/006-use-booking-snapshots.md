# ADR-006: Store Booking Snapshots

## Status

Accepted

---

## Context

Customer information and provider data may change over time.

Examples include:

- Passenger names.
- Passport information.
- Hotel room names.
- Flight schedules.
- Cancellation policies.
- Prices.

A confirmed booking must preserve the exact information that existed at booking time.

The booking should remain historically accurate even if source data changes later.

---

## Decision

The platform stores snapshot data inside booking-related tables.

Examples include:

Flight Booking

- Passenger information
- Flight segments
- Fare information

Hotel Booking

- Guest information
- Room information
- Cancellation policy

Snapshot data represents the booking at confirmation time and is never automatically synchronized with provider updates.

---

## Alternatives Considered

### Reference Live Data Only

Store only IDs and always retrieve current provider data.

**Pros**

- Less storage.
- No duplicated data.

**Cons**

- Historical data becomes inaccurate.
- Provider changes affect old bookings.

---

### Partial Snapshot

Snapshot only a few important fields.

**Pros**

- Less storage.

**Cons**

- Inconsistent historical records.
- Missing business information.

---

## Consequences

### Positive

- Accurate historical records.
- Better auditing.
- Stable booking confirmations.
- Independent of future provider changes.

### Negative

- Larger database size.
- Controlled data duplication.

---

## Related Documents

- `09-booking-orchestration.md`
- `12-database-design.md`