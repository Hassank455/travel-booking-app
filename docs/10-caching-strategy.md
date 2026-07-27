# 10. Caching Strategy

## Purpose

This document defines the caching strategy for the Travel Booking Platform.

Its objective is to improve platform performance, reduce unnecessary communication with external providers, and increase scalability while ensuring that business correctness is never compromised.

Caching is treated as a performance optimization layer rather than a persistence mechanism or source of truth.

---

## Scope

This document covers the caching strategy used across the platform, including:

- Search result caching
- Static reference data caching
- Provider metadata caching
- Session-related caching
- Cache expiration and invalidation
- Cache consistency principles
- Failure handling

This document does not cover:

- Redis deployment configuration
- Infrastructure provisioning
- Database persistence
- Provider-specific implementation details

---

## Why Caching Exists

Travel booking platforms communicate with multiple external providers whose responses may introduce latency and usage costs.

Without an effective caching strategy, the platform would repeatedly request identical information, increasing response time and unnecessary provider traffic.

The caching strategy exists to:

- Reduce search latency.
- Reduce calls to external providers.
- Improve overall platform scalability.
- Lower infrastructure and provider costs.
- Improve user experience.
- Reduce duplicate processing.

Caching improves performance but never replaces business validation.

---

## Caching Principles

The platform follows several architectural principles regarding cache usage.

### Cache is Never the Source of Truth

Business-critical information is always validated against the authoritative source before making business decisions.

---

### Business Correctness Comes First

Performance optimizations must never compromise booking accuracy.

---

### Revalidation Before Booking

Cached search results are temporary.

Every booking request must revalidate the selected offer before confirmation.

---

### Cache is Disposable

The platform must continue operating even if cached data becomes unavailable.

---

### Cache Lifetime Depends on Business Freshness

Frequently changing information should expire sooner than relatively static information.

---

### Graceful Degradation

If the cache becomes unavailable, the platform should continue operating using external providers, even if response time increases.

---

## Data Classification

Different types of data require different caching strategies.

Instead of applying the same cache policy to every dataset, the platform classifies data based on business characteristics such as volatility, update frequency, and business criticality.

---

### Dynamic Business Data

Dynamic data changes frequently and directly affects customer decisions.

Examples include:

- Flight search results
- Hotel search results
- Prices
- Availability
- Promotional offers

Characteristics:

- Frequently updated
- Short-lived
- Retrieved from external providers
- Must be revalidated before booking

Although this data is cached to improve search performance, it should never be trusted without revalidation during the booking process.

---

### Static Reference Data

Reference data changes infrequently and is shared across the platform.

Examples include:

- Airports
- Airlines
- Countries
- Cities
- Currencies
- Hotel amenities
- Cabin classes
- Meal types

Characteristics:

- Rarely updated
- Long cache lifetime
- Shared by many services
- Safe to cache

This type of data provides an excellent caching opportunity because it changes infrequently while being requested frequently.

---

### Provider Metadata

Provider metadata describes the capabilities and configuration of external providers rather than customer-facing business data.

Examples include:

- Supported countries
- Supported currencies
- Available APIs
- Provider capabilities
- Feature availability
- Timeout configuration

Characteristics:

- Changes occasionally
- Medium cache lifetime
- Shared across requests

Caching provider metadata reduces unnecessary configuration lookups and simplifies provider selection.

---

### Critical Transactional Data

Transactional data represents business operations that directly impact customers.

Examples include:

- Bookings
- Payments
- Transactions
- Refunds
- Cancellations

Characteristics:

- Business critical
- Frequently modified
- Requires strong consistency

Transactional data must never use cache as its authoritative source.

All reads and writes must rely on the primary database.

---

## Cached Data Types

| Data Type | Cache | Cache Lifetime | Source of Truth |
|-----------|:-----:|----------------|-----------------|
| Flight Search Results | ✅ | Short | External Providers |
| Hotel Search Results | ✅ | Short | External Providers |
| Airport List | ✅ | Long | Database |
| Airline List | ✅ | Long | Database |
| Country List | ✅ | Long | Database |
| Currency List | ✅ | Long | Database |
| Hotel Amenities | ✅ | Long | Database |
| Provider Metadata | ✅ | Medium | Provider Configuration |
| User Session | ✅ | Short | Authentication Service |
| Booking | ❌ | Never | Database |
| Payment | ❌ | Never | Database |
| Transaction | ❌ | Never | Database |
| Refund | ❌ | Never | Database |

---

## Cache Layers

The platform uses multiple layers of caching, each serving a different purpose.

Rather than relying on a single cache, each layer optimizes a specific part of the request lifecycle while maintaining clear separation of responsibilities.

---

### Layer 1 — Application Memory

The application maintains short-lived in-memory data that is only relevant to the running service instance.

Typical use cases include:

- Frequently accessed configuration
- Provider clients
- Parsed configuration files
- Temporary lookup objects

Characteristics:

- Fastest access
- Local to a single application instance
- Lost when the service restarts

---

### Layer 2 — Distributed Cache

A distributed cache (such as Redis) stores data shared across all application instances.

Typical use cases include:

- Search results
- User sessions
- Reference data
- Provider metadata

Characteristics:

- Shared across the platform
- High performance
- Independent from application instances
- Automatically expires based on business rules

---

### Layer 3 — Primary Database

The primary database is the authoritative source for business data.

Examples include:

- Customers
- Bookings
- Payments
- Transactions

The database guarantees consistency and durability.

Unlike cache, data stored here is considered permanent until explicitly modified.

---

### Layer 4 — External Providers

External providers remain the ultimate source for real-time travel inventory.

Examples include:

- Flight providers
- Hotel providers
- Availability services
- Pricing services

Provider responses may be temporarily cached to improve performance, but they remain the authoritative source until a booking is confirmed.

---

## Cache Architecture

The following diagram illustrates how requests travel through the different cache layers before reaching the external providers.

**Diagram**

[Cache Architecture](./diagrams/cache/cache-architecture.mmd)

---

## Cache Lookup Strategy

Whenever possible, the platform attempts to retrieve data from the fastest available layer.

The lookup order follows a cache-first approach.

1. Check application memory.
2. Check distributed cache.
3. Query the database when applicable.
4. Query external providers if no cached data exists.
5. Store eligible data back into the cache.

This strategy minimizes unnecessary provider requests while ensuring that business-critical information remains accurate.

---

## Cache Population

The platform populates cache using a lazy-loading strategy.

Cache entries are created only when data is requested.

If the requested data does not exist in cache:

- Retrieve data from the authoritative source.
- Normalize the response if necessary.
- Store eligible data in cache.
- Return the response to the client.

This approach prevents unnecessary cache growth and avoids storing unused data.

---

## Cache Key Design

Cache keys must be generated from normalized business parameters rather than raw request payloads.

The platform uses versioned and deterministic cache keys for:

- Aggregated flight and hotel searches
- Provider-level search responses
- Reference data
- Sessions and temporary operational data

Search cache key normalization, hashing, provider-level caching, provider selection strategy, pagination, localization, and currency considerations are documented separately.

For full details, see:

[Cache Key Design](./11-cache-key-design.md)

---

## TTL Strategy

Cache lifetime should reflect business freshness rather than technical convenience.

Different categories of data change at different rates and therefore require different cache lifetimes.

Instead of applying a single expiration policy across the platform, each cached dataset should use a TTL that matches its business characteristics.

---

### Very Short Lifetime

Data that changes rapidly should expire quickly.

Examples include:

- Flight availability
- Hotel availability
- Dynamic pricing
- Promotional offers

These datasets become outdated quickly and therefore require aggressive expiration.

---

### Short Lifetime

Frequently requested data that changes regularly may remain cached for a short period.

Examples include:

- Flight search results
- Hotel search results

This improves search performance while keeping results reasonably fresh.

---

### Medium Lifetime

Data that changes occasionally can remain cached longer.

Examples include:

- Provider metadata
- Supported destinations
- Provider capabilities

These datasets rarely change during normal operations.

---

### Long Lifetime

Reference data that rarely changes can have significantly longer cache lifetimes.

Examples include:

- Airports
- Countries
- Airlines
- Currencies
- Hotel amenities

Long-lived cache entries reduce unnecessary database queries while maintaining accuracy.

---

## Cache Invalidation

Cache invalidation ensures that outdated information is removed before it affects business operations.

Rather than relying solely on expiration, the platform invalidates cache whenever significant business events occur.

---

### Search Cache Expiration

Search cache becomes invalid when:

- Cached results expire.
- Provider availability changes.
- Pricing changes significantly.

The next search rebuilds the cache using fresh provider data.

---

### Provider Configuration Changes

Whenever provider capabilities or configuration are updated, related metadata cache should be invalidated.

This ensures future requests use the latest provider configuration.

---

### Reference Data Updates

When administrative users update static reference data such as airports or countries, the corresponding cache entries should be refreshed.

---

### Manual Invalidation

Platform administrators may invalidate cache manually during:

- Provider maintenance
- Emergency fixes
- Configuration updates
- Operational troubleshooting

Manual invalidation should affect only the necessary cache entries whenever possible.

---

## Cache Consistency

The platform prioritizes business correctness over cache consistency.

Temporary inconsistencies between cache and authoritative data are acceptable as long as they cannot lead to incorrect business outcomes.

To maintain consistency:

- Cached offers are always revalidated before booking.
- Booking operations never rely exclusively on cached data.
- Payments always use the primary database.
- Critical business operations bypass cache when necessary.

This approach provides high performance while preserving transactional correctness.

---

## Failure Strategy

The cache is considered an optimization layer rather than a required dependency.

If the cache becomes unavailable, the platform should continue operating.

Expected behavior includes:

- Search requests continue by querying external providers.
- Booking operations continue using authoritative data.
- Payments remain fully operational.
- User experience may degrade due to increased response time.

The platform should recover automatically once the cache becomes available again.

---

## Security Considerations

Only non-sensitive and cache-eligible data should be stored in cache.

The platform should avoid storing:

- Payment credentials
- Authentication secrets
- Personally identifiable information when unnecessary
- Sensitive financial information

Cache entries should expire automatically according to business requirements.

Administrative cache operations should be restricted to authorized users.

---

## Architectural Decisions

The caching strategy follows the following architectural decisions:

- Cache exists to improve performance, not to replace the primary database.
- Business correctness always takes precedence over cache efficiency.
- Search results are temporary and require revalidation before booking.
- Cache lifetime is determined by business freshness requirements.
- Critical transactional data is never treated as cache-authoritative.
- Cache failures must not interrupt booking or payment operations.
- Cache keys are generated from normalized business requests.
- Cache invalidation is event-driven whenever possible.
- The platform should degrade gracefully when cache is unavailable.

---

## Related Diagrams

- [Cache Overview](./diagrams/cache/cache-overview.mmd)
- [Cache Architecture](./diagrams/cache/cache-architecture.mmd)
- [Cache Lifecycle](./diagrams/cache/cache-lifecycle.mmd)
- [Cache Invalidation](./diagrams/cache/cache-invalidation.mmd)
- [Search Cache Flow](./diagrams/cache/search-cache-flow.mmd)

---

## Related Documents

- [08. Search Architecture](./08-search-architecture.md)
- [09. Booking Orchestration](./09-booking-orchestration.md)
- [11. Provider Integration](./11-provider-integration.md)