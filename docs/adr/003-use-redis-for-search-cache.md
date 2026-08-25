# ADR-003: Use Redis for Search Cache

## Status

Accepted

## Context

Flight and hotel searches depend on external provider APIs.

External searches may:

- Introduce significant latency.
- Consume provider rate limits.
- Increase provider API costs.
- Produce repeated results for identical customer searches.

The platform is expected to receive repeated search requests with equivalent parameters.

Calling all providers for every request would unnecessarily increase latency and external traffic.

Search results are also temporary by nature and do not belong in permanent PostgreSQL storage.

## Decision

The platform will use Redis as a distributed cache for temporary search results.

Search requests will first be normalized and converted into deterministic cache keys.

Conceptually:

```text
Search Request
      ↓
Normalize Request
      ↓
Generate Cache Key
      ↓
Redis
   ┌──┴──┐
 Hit    Miss
  │       │
Return   Providers
          ↓
      Normalize
          ↓
       Cache
```

Redis may also support related short-lived technical state such as:

- Idempotency records.
- Distributed locks.
- Temporary orchestration state.

Redis will not become the authoritative source for bookings or transactions.

## Alternatives Considered

### No Search Cache

Every search request could call external providers directly.

**Pros**

- Maximum data freshness.
- Simplest consistency model.

**Cons**

- Higher latency.
- More provider traffic.
- Higher external API cost.
- Increased exposure to provider rate limits.

### PostgreSQL as Search Cache

Temporary search results could be persisted in PostgreSQL.

**Pros**

- Durable storage.
- Existing infrastructure.

**Cons**

- Search offers are temporary.
- Creates unnecessary database writes.
- Requires cleanup and expiration management.
- Mixes transactional persistence with temporary search state.

### Application Memory Cache

Each backend instance could cache results locally.

**Pros**

- Extremely fast.
- Simple.

**Cons**

- Cache is not shared across horizontally scaled instances.
- Duplicate provider calls may occur across instances.
- Cache is lost when the application restarts.

## Consequences

### Positive

- Lower search latency.
- Fewer external provider requests.
- Better provider quota utilization.
- Shared cache across application instances.
- Natural support for TTL-based expiration.

### Negative

- Additional infrastructure dependency.
- Cache invalidation and expiration policies must be managed.
- Temporary stale data is possible.
- Cache stampede protection may be required.

## Related Documents

- `08-search-architecture.md`
- `10-caching-strategy.md`
- `cache-key-design.md`
- `13-security.md`
- `14-observability.md`