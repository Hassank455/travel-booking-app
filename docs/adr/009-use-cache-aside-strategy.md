# ADR-009: Use Cache-Aside Strategy

## Status

Accepted

---

## Context

Flight and hotel searches are read-heavy operations that depend on external provider APIs.

Calling external providers for every identical search would:

- Increase response latency.
- Consume provider rate limits.
- Increase infrastructure and provider costs.
- Reduce overall system performance.

The platform requires a caching strategy that improves performance while keeping external providers as the authoritative source of travel inventory.

---

## Decision

The platform adopts the **Cache-Aside** pattern for search caching.

The application is responsible for interacting with the cache.

The flow is:

```text
Search Request
      ↓
Check Redis Cache
      ↓
 ┌───────────────┐
 │               │
Hit            Miss
 │               │
Return      Call Providers
                 ↓
          Normalize Results
                 ↓
           Store in Redis
                 ↓
             Return Results
```

Redis acts as a temporary cache only.

External providers remain the authoritative source for flight availability, hotel availability, pricing, and offers.

---

## Alternatives Considered

### No Cache

Always call external providers.

**Pros**

- Always returns the freshest data.
- Simple implementation.

**Cons**

- High response latency.
- Increased provider costs.
- More provider rate-limit issues.
- Poor user experience.

---

### Write-Through Cache

Update the cache whenever the underlying data changes.

**Pros**

- Cache remains fresh.
- Reads are very fast.

**Cons**

- Not practical because provider data is owned externally.
- The platform cannot detect every provider-side update.
- Adds unnecessary complexity.

---

### Read-Through Cache

Let the cache automatically load missing data.

**Pros**

- Simple application code.
- Cache manages data loading.

**Cons**

- Less control over provider calls.
- Harder to implement provider-specific logic.
- Less flexible for multi-provider aggregation.

---

## Consequences

### Positive

- Reduced response time.
- Lower provider API usage.
- Better cache hit ratio.
- Lower operational cost.
- Simple and widely adopted pattern.

### Negative

- First request after cache expiration is slower.
- Cache invalidation must be carefully managed.
- Cache stampede protection may be required.

---

## Related Documents

- `08-search-architecture.md`
- `10-caching-strategy.md`
- `cache-key-design.md`
- `14-observability.md`