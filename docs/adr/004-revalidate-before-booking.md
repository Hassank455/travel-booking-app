# ADR-004: Revalidate Offers Before Booking

## Status

Accepted

## Context

Flight and hotel search offers represent provider availability and pricing at the time the search was performed.

Between search and booking:

- Prices may change.
- Flight seats may become unavailable.
- Hotel rooms may become unavailable.
- Fare rules may change.
- Cancellation conditions may change.
- Temporary provider offers may expire.

Therefore, a search result cannot be treated as guaranteed booking inventory.

This is especially important because search results may also be returned from Redis cache.

## Decision

Every selected offer must be revalidated before the platform attempts final booking confirmation.

Conceptually:

```text
Search
   ↓
Offer
   ↓
Customer Selects Offer
   ↓
Revalidation
   ↓
Valid?
 ┌─┴─┐
No  Yes
│     │
Return Updated
Information
      ↓
Booking Process
```

For flights, the provider adapter will retrieve or validate the current offer before booking.

For hotels, the equivalent provider-specific validation step may include a prebook or rate-validation operation.

The Booking Module owns this rule.

Cached search data never bypasses revalidation.

## Alternatives Considered

### Book Directly from Search Result

The platform could use the original search result without revalidation.

**Pros**

- Lower booking latency.
- Fewer provider calls.

**Cons**

- Customer may be charged an outdated price.
- Availability may no longer exist.
- Higher provider booking failure rate.
- Incorrect cancellation or fare rules may be used.

### Very Short Cache TTL Without Revalidation

The platform could rely on an extremely short search cache TTL.

**Pros**

- Reduces stale-data probability.
- Simple booking flow.

**Cons**

- Fresh cache does not guarantee provider availability.
- Inventory may change immediately after search.
- Provider contracts may still require explicit validation.

## Consequences

### Positive

- Reduces stale-offer booking failures.
- Protects booking correctness.
- Allows search caching without trusting cached offers for transactions.
- Ensures customers confirm current commercial terms.

### Negative

- Adds an additional provider request.
- Slightly increases booking latency.
- Price changes may require additional customer confirmation.
- Booking orchestration becomes more complex.

## Related Documents

- `08-search-architecture.md`
- `09-booking-orchestration.md`
- `10-caching-strategy.md`
- `12-database-design.md`