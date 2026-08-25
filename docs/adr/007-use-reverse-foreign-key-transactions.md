# ADR-007: Use Reverse Foreign Key for Booking Transactions

## Status

Accepted

---

## Context

The platform stores flight bookings and hotel bookings in separate tables.

Each booking currently has exactly one financial transaction.

Several relationship designs were considered:

1. Multiple nullable foreign keys inside the `transactions` table.
2. A polymorphic association (`booking_type` + `booking_id`).
3. A transaction foreign key stored inside each booking table.

Because transactions represent financial records, maintaining strong database referential integrity is essential.

---

## Decision

Each booking table will store a foreign key referencing its transaction.

Example:

```text
flight_bookings.transaction_id
        ↓
transactions.id
```

```text
hotel_bookings.transaction_id
        ↓
transactions.id
```

Each booking owns exactly one transaction.

The `transactions` table remains independent and does not contain booking-type-specific foreign keys.

---

## Alternatives Considered

### Option 1 — Multiple Foreign Keys in Transactions

Example:

```text
transactions

flight_booking_id
hotel_booking_id
```

**Pros**

- Strong referential integrity.
- Native foreign keys.

**Cons**

- Nullable columns.
- Requires schema changes whenever a new booking type is introduced.
- Couples the transaction entity to every booking domain.

---

### Option 2 — Polymorphic Association

Example:

```text
booking_type
booking_id
```

**Pros**

- Flexible.
- Easy to extend with future booking types.

**Cons**

- Database cannot enforce referential integrity.
- Validation depends on application logic.
- Not ideal for financial relationships.

---

## Consequences

### Positive

- Strong database integrity.
- Clear ownership between booking and transaction.
- No nullable foreign keys.
- Easy to support additional booking types.

### Negative

- Every booking table must include a `transaction_id`.
- Queries across all booking types require accessing multiple tables.

---

## Related Documents

- `09-booking-orchestration.md`
- `12-database-design.md`