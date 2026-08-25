# ADR-011: Redis Is Not the Source of Truth

## Status

Accepted

---

## Context

The platform uses Redis to improve performance by storing temporary and frequently accessed data.

Redis is used for:

- Search cache.
- Idempotency records.
- Distributed locks.
- Temporary orchestration state.

However, the platform also manages critical business data such as:

- Flight bookings.
- Hotel bookings.
- Transactions.
- Customers.

This data requires strong consistency, durability, and transactional guarantees.

Redis is an in-memory datastore and should not be treated as the permanent owner of business data.

---

## Decision

Redis will be used exclusively as a temporary infrastructure component.

It will never be considered the authoritative source of business data.

The authoritative sources are:

### PostgreSQL

Owns:

- Customers
- Travelers
- Flight Bookings
- Hotel Bookings
- Transactions

### External Providers

Own:

- Flight availability
- Hotel availability
- Prices
- Offers

Redis only stores temporary copies of this data when appropriate.

Conceptually:

```text
Customer Search
        ↓
Redis Cache
   ↓           ↓
 Hit         Miss
  │            │
Return    External Providers
                ↓
          Normalize Results
                ↓
             Redis
                ↓
             Customer
```

If Redis becomes unavailable, the platform should continue operating by retrieving data from the authoritative source.

---

## Alternatives Considered

### Redis as Primary Data Store

Store business data directly in Redis.

**Pros**

- Extremely fast reads.
- Simple lookup operations.

**Cons**

- Business data becomes dependent on cache availability.
- Increased risk of data loss.
- Weak transactional guarantees.
- Blurs the distinction between cache and persistence.

---

### Database Only

Do not use Redis.

**Pros**

- Simple architecture.
- Single authoritative data source.

**Cons**

- Increased response latency.
- Higher database load.
- More external provider requests.
- Poor scalability for read-heavy workloads.

---

## Consequences

### Positive

- Clear separation between persistence and caching.
- Redis failures affect performance rather than data correctness.
- Cache entries can be safely rebuilt.
- Simpler recovery procedures.

### Negative

- Cache misses require additional database or provider requests.
- Multiple authoritative sources must be understood by developers.

---

## Related Documents

- `10-caching-strategy.md`
- `12-database-design.md`
- `13-security.md`
- `14-observability.md`