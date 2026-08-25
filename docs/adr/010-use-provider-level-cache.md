# ADR-010: Introduce Provider-Level Cache When Needed

## Status

Proposed

---

## Context

The platform integrates with multiple external providers for flight and hotel searches.

Each provider may have different:

- Response latency.
- Rate limits.
- Pricing models.
- Availability.
- Cache freshness requirements.
- Failure characteristics.

The current architecture uses an Aggregated Search Cache, where the final combined search results are cached.

When an aggregated cache entry expires, the platform must call every provider again to rebuild the result.

For high traffic or many providers, this may become inefficient.

---

## Decision

The platform will initially use only the Aggregated Search Cache.

If operational requirements justify additional optimization, a Provider-Level Cache may be introduced.

Each provider will maintain its own independent cache entries.

Example:

```text
Duffel

flight-provider:duffel:{requestHash}

LiteAPI

hotel-provider:liteapi:{requestHash}
```

Conceptually:

```text
Search Request
        ↓
Aggregated Cache
        ↓
      Miss
        ↓
 ┌───────────────┐
 │               │
Duffel Cache   LiteAPI Cache
 │               │
Hit            Miss
 │               │
Reuse       Call LiteAPI
        ↓
Aggregate Results
        ↓
Update Aggregated Cache
```

This approach allows individual provider responses to be reused independently.

---

## Alternatives Considered

### Aggregated Cache Only

**Pros**

- Very simple implementation.
- Easy cache invalidation.
- Fast response for repeated identical searches.
- Lower operational complexity.

**Cons**

- A cache miss requires querying every provider again.
- Cannot reuse individual provider responses.

---

### Provider-Level Cache Only

**Pros**

- Independent cache per provider.
- Better reuse of provider responses.
- Flexible provider-specific TTLs.

**Cons**

- Every request still requires result aggregation.
- More cache lookups.
- More complex implementation.

---

## Consequences

### Positive

If adopted:

- Fewer provider API calls.
- Better utilization of provider quotas.
- Independent cache expiration policies.
- Better resilience when one provider is slower than others.

### Negative

- Increased implementation complexity.
- Additional cache management.
- More complicated cache invalidation.
- Larger Redis footprint.

---

## Adoption Criteria

Provider-Level Cache should be introduced only if one or more of the following conditions are met:

- Provider API costs become significant.
- Provider rate limits frequently impact searches.
- Search traffic increases substantially.
- Providers require different cache expiration policies.
- Monitoring demonstrates measurable performance benefits.

Until these conditions exist, the simpler Aggregated Search Cache remains the preferred approach.

---

## Related Documents

- `08-search-architecture.md`
- `10-caching-strategy.md`
- `cache-key-design.md`
- `14-observability.md`